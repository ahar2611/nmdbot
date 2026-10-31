# CLAUDE.md: Ninja Must Die Daily Bounty Bot

Read this whole file before writing any code. Keep a `PROGRESS.md` next to it (what is done, what is next, what failed and why) and update it at the end of every session, so a fresh session can pick up without re-asking questions.

---

## 0. How to work on this project

- Build in small, testable steps (section 13). Do not generate the whole bot at once. Finish a step, run it on the real phone, then move on. This keeps token usage low and bugs local.
- Anything marked **VERIFY** or listed in section 15 (open questions) is not known yet. Do not guess. Build the part that does not depend on it, put a clearly named config value or TODO where it goes, and ask the user for a screenshot or answer.
- Coordinates in this file are **approximate**, measured on screenshots from the user's phone. Re-derive them by cropping templates from real frames (section 14). Prefer template matching over hard-coded coordinates wherever a button can move.
- The user does not review the generated code line by line, so the safety rules in section 10 are mandatory and must exist before the bot ever sends a tap.
- Always provide a **dry-run mode** (logs the tap it would send but sends nothing) and use it for the first run of every new flow.

---

## 1. Goal

Automate the daily **Bounty** chores in the Android game **Ninja Must Die** (the user's account is the player named "A1"), so the user does not have to do them by hand.

Per day the bot should, in this order:
1. Read which bounty ranks still need to be completed (the Member Reward Limits panel).
2. Claim the **free daily bounty** from the Bounty Shop (and buy nothing).
3. **Host first:** run **all of the user's owned daily bounties** from the Bounty screen carousel, in **no particular order** (user decision), recruiting other players through the Team Recruitment chat. Only **daily ranks** (D, C, B, A, A+) are hosted. Owned bounties of weekly ranks (S, S+, SS, SS+) are **skipped**. Hosting does not depend on the participant cap: the owner always gets the owner reward, so every owned daily bounty is run.
4. **Then look in the chat for what is still missing:** join other players' bounties posted in the public Team Recruitment chat, only for **daily ranks that are still below their daily cap** after hosting (read the Member Reward Limits panel again after the hosting phase).
5. Stop when every enabled rank has reached its cap (or a safety limit is hit) and report what happened.

**Reward logic (from the user):** every bounty has an owner reward and a participant reward. The owner always gets the owner reward, plus the participant reward if the owner has not reached the daily participant cap. So running your own bounties is always better than joining someone else's, which is why hosting comes first.

**Out of scope:** actually playing bosses skillfully. The daily battles **cannot be lost** (user statement). The only in-battle behavior needed is described in section 6.11.

---

## 2. Environment

- PC: **Windows**. Phone: **Android**, connected to the PC by **USB cable**. No emulator.
- Python 3.11+, with: `opencv-python`, `numpy`, `mss` (and/or `dxcam`), `pyautogui` or `pywin32` (window clicks), `pyyaml`, `keyboard` (kill hotkey), `pytesseract` or `easyocr` (optional, section 4.4).
- **scrcpy** mirrors the phone to a PC window with low latency. Enable Developer options, then USB debugging, on the phone.
  - Suggested launch: `scrcpy --no-audio --max-fps 60 --window-x 0 --window-y 0 --window-width 1288 --window-height 594` (adjust to the phone's aspect ratio; the game is **landscape**).
- **Capture:** grab the scrcpy window region directly (`dxcam` or `mss`, a few ms per frame). Do not use `adb screencap` for the main loop (about 0.5 to 1 s per frame).
- **Input:** clicks sent to the scrcpy window act as touches. Use `pywin32`/`pyautogui` clicks on the window. `adb shell input tap` is a **fallback only**, it adds 100 to 300 ms per tap.
- The scrcpy window must stay visible, not minimized, at a fixed position and size. The bot should verify the window rectangle at startup and on every few frames, and stop if it changes.
- Any reasonable video stream works (the reference bot also works over screen sharing), but scrcpy over cable is the supported setup here.

---

## 3. Requirements

### 3.1 Speed
- Bounties in the public chat fill **extremely fast**, a few seconds of delay is too slow.
- Target: from a new bounty card appearing to the Join tap in **under about 300 ms**. The reference bot's navigation reacts in 10 to 100 ms, so this is realistic.
- Do not use cloud/LLM calls or heavy OCR in the fast path. Use template matching and color checks on small regions of interest (ROIs), on a **downscaled frame**.
- Capture at 30+ fps. Template match per ROI should be roughly 5 to 15 ms.
- Re-detect right before tapping. Never reuse button coordinates from an earlier frame, because the chat list shifts when new bounties are posted.

### 3.2 Reliability
- A reference bot by another player took about a month to make reliable. Expect many `unknown` screens and transition screens early on. Design the bot around **logging and replay** so each failure can be fixed from saved evidence (section 11).
- When something is unrecognized for too long, stop tapping and notify the user (section 9.4).

### 3.3 Human-likeness
- The reference bot's author once made it so fast it "reacted too fast" and reverted that. Add small random jitter (about 60 to 250 ms) to **non-time-critical** taps (menu navigation, Confirm). The chat Join tap should still be as fast as possible.
- Randomize tap positions slightly inside the button (a few pixels), not always the exact center.

---

## 4. Architecture

A **screen-state machine**, modeled on the reference bot's log vocabulary:

- **View**: which screen the game is on (identified by template matching, with a confidence score). Examples: `main`, `chats/recruitment-chat`, `bounty-owned`, `bounty-shop`, `lobby/host`, `lobby/joined`, `lobby/joined-ready`, `battle-loading`, `battle`, `battle-warning`, `rewards-popup`, `results`, `unknown`.
- **Action**: one tap or gesture with an expected outcome view. Examples: `open-chats`, `join-bounty(rank, index)`, `ready`, `go`, `send-invite`, `confirm`.
- **Flow**: a chain of actions toward a goal, with a clear success and failure result. Examples: `claim-free-bounty`, `read-limits`, `host-bounty`, `join-bounty`, `battle`, `finish-results`.

Main loop (single thread for vision plus input, so state is never stale):
1. Capture a frame.
2. Identify the view (cheap checks first, then the specific templates for the views that are possible from the current state).
3. Ask the active flow for the next action.
4. Send the action, then wait for the **expected next view** within a grace window (section 9.2).
5. Log everything (section 11).

### 4.1 View detection
- Each view has a small set of **anchor templates** (static text or icons) plus optional color checks. Combine them into a confidence score (0.0 to 1.0).
- Match only on **static lettering and icons**. **Mask or avoid** anything that changes: countdown timers, red-dot badges, chat badge counts, currency numbers, event banners (the main-screen banner changes), character art in lobbies, player names.
- Use the **previous view and the expected next views** to narrow what to test, which also makes detection faster.
- `unknown` is a valid result. A single `unknown` frame during a transition is normal (section 9.2).

### 4.2 Resolution handling
- Reference frame for all coordinates in this file: **2576 x 1189** (the phone's screenshot size). Store coordinates as ratios (x/2576, y/1189) in config, and resize frames to a fixed working size for matching (for example half size, 1288 x 594).
- Templates are cropped at the working size. If the scrcpy window size changes, templates must be regenerated or scaled consistently.

### 4.3 Local state
Keep a small `state.json` with:
- Per-rank counters (what has been done today), per-rank caps, and the date they were last verified.
- Whether the free bounty has been claimed today.
- Last invite time, last results seen, and totals for the run.
Counters are updated from the **results screen title** after each bounty and **verified** against the Reward Limits panel at startup and every few bounties (the panel is the source of truth).

### 4.4 OCR
Used only in non-time-critical places, for example reading the `x/y` numbers in the limits panel if digit templates are not enough, or double-checking progress. Never in the chat-join fast path.

---

## 5. Game facts

### 5.1 Ranks and caps
Bounty ranks: **D, C, B, A, A+, S, S+, SS, SS+**. The Reward Limits panel (example from the user's account) shows:
- Daily: D 0/5, C 0/5, B 0/4, A/A+ 0/3 (A and A+ share one counter).
- Weekly: S/S+ 1/5, SS 0/1, SS+ 1/1.
Whether the bot should also handle the weekly S-ranks is **open** (section 15). Default: **off**, controlled by `enabled_ranks` in config.

Boss names seen so far per rank (from chat cards, the carousel and lobbies): **D**: Big Bat, Black Wings (lv.30), Red Face (lv.30). **C**: Giant Bat, Green Face, Shikigami - Wicked, Thunder Dragon (lv.40). **B**: Lightning Wings (lv.60), Crimson Flame Sword (lv.60), Vicious Green Face. **A**: Purple Face (lv.80). **S**: White Face. Read the **rank letter** in the header, not the boss, to decide whether a card is wanted.

### 5.2 Bounty Shop
- Contains a **Free Bounty** card (random rank, shown as "?") that can be claimed once per cooldown (about once per day), and **priced** bounty cards (D 3000, C 10000, and more further right).
- **The bot must never buy anything.** It only claims the Free Bounty.

### 5.3 Hosting vs joining
- Hosting: open the Bounty screen, tap a bounty card in the carousel (when it is centered), a short animation plays, then the **host lobby** opens. Use **Team Invite** to post it to the Team Recruitment channel, wait for joiners.
- Joining: in the chat, tap **Join** on someone else's card, then press **Ready**, the host presses Go, battle starts.
- Lobby sizes: C bounties 3 slots, A and S bounties 5 slots (examples seen). Minimum 2 players to start ("The team needs at least 2 players").

---

## 6. Screen catalog

All coordinates are in the 2576 x 1189 reference frame and **approximate**.

### 6.1 Main screen (`main`)
- Landscape home screen with the player's ninja in the middle. Icons down the left (Battle Pass, Ninja Trials, Friends, Archive, Chat, Ninja Cultivation, Backpack, Four Symbols Exploration, Summons, Abyss Challenge). Row along the bottom (Ninja, Team, Weapon, Relic, Arcana, Shop, Wish). Large tiles on the right: Story, **Bounty**, Race Mode, Clan, Arena.
- Anchors: the lettering "Bounty", "Story", "Race Mode", "Arena", "Clan" on the tiles, and the "Chat" icon at left.
- Bounty tile: about (1790, 760) (tile spans roughly x 1610 to 1960, y 660 to 850). It shows a **countdown** ("03h59m") and a red dot: mask both.
- Chat icon: about (196, 670).
- The event banner (top right, about x 1500 to 2350, y 180 to 400) **changes**. Never match on it. A "Click to join official Discord" bubble can appear over the lower middle of the screen when the chat is open (see 6.2).

### 6.2 Chat panel, Team Recruitment tab (`chats/recruitment-chat`)
- A panel covering the **left half** of the screen (about x 0 to 1312). Tabs along the top: Chat, Clan, North America, **Team Recruitment** (tab center about (1160, 66)). A filter (funnel) icon sits at about (1365, 160) on the right edge of the panel. Tab badges (numbers) change, so mask them.
- Cards stack vertically. **Newest is at the bottom** and the list **auto-scrolls**, so old cards only show up when nothing new has been posted for a while.
- Each card (x about 280 to 1120):
  - Poster name and time at the top (ignore).
  - Round **boss portrait** (about x 320 to 505).
  - Blue header strip with **rank letter + boss name** ("C Bounty   Giant Bat", "A Bounty   Purple Face", "S Bounty   White Face", "D Bounty   Big Bat", "B Bounty   Vicious Green Face").
  - Gray strip "Minimum Rank  Senior Ninja".
  - "Team Members: 1/3" (or 1/5, 2/5).
  - A yellow **Join** button on the lower right (button spans about x 805 to 1075).
- Vertical layout is consistent: the header strip center is about **238 px above the Join button center**, the boss portrait center about 190 px above it. Use this to pair a Join button with its header. Card heights vary (a card with a title badge or fire icon is taller), so do not assume fixed spacing between cards.
- **Expired cards** are grayed out, show "Expired" where the Join button would be, and have **no yellow Join**. The bot ignores them. The bot does **not** need to detect "full" cards: cards do not update instantly, so a stale or full card may still show Join. Handle this by verifying the result of each Join tap (section 9.2).
- Overlay: the Discord bubble ("Click to join official Discord>>") appears bottom-middle, **outside** the panel on the right. It does not block the chat but must never be tapped.
- To close the panel, tap outside it or use the game's back path. VERIFY exactly how.

### 6.3 Bounty screen (`bounty-owned`)
- Title "Bounty" with a back arrow at about (212, 82). Top right: **Training House** (about (2005, 100)), **Shattered Recipe Shop** (about (2200, 100)), **Bounty Shop** (about (2390, 100)), each possibly with a red dot.
- A **carousel of the user's owned bounties**. The centered card is large: rank ("B Bounty", "C Bounty"), a **countdown** ("03h59m left"), boss art, boss name and level ("Lightning Wings lv.60", "Giant Bat lv.40"), and a reward line ("Obtain random B-S Relic"). Side cards are narrower silhouettes showing rank, boss name, level. Match only on the **centered card** region and mask the countdown.
- Centered card region: about x 1015 to 1560, y 215 to 1140. Tapping it (when centered) starts the host flow (animation, then host lobby).
- **Member Reward Limits** panel: a chest icon with a label at the bottom right (about (2230, 1112)) and a chevron at about (2395, 1112). The chevron points up when collapsed and down when expanded. Expanded, it shows the Daily and Weekly counters (examples in 5.1) in a panel above the label. Read counters from this panel.
- Bottom left: chat bubble (about (225, 1110), with a changing badge) and Online Friends. The chat bubble can be used to open the chat from this screen.
- **How the carousel works (second recording, section 18b):** **tapping the side card right next to the middle brings it to the center** (only the adjacent card responds, a card two places away does not jump, so scroll one card per tap) (animation about 0.3 to 0.5 s, the card scales up and the others shift). Tapping the **centered** card opens the host lobby (about 0.4 s of black, then the lobby about 1.0 s after the tap). The carousel shows owned bounties of several ranks at once (for example D "Black Wings" lv.30, D "Red Face" lv.30, C "Thunder Dragon" lv.40). The countdown can read days ("1d05h left", "2d20h left"), so mask it. After a hosted bounty is finished, the game returns to this screen with the carousel re-centered on the **first** remaining card, and the finished bounty is gone. Hosting order is **no particular order, daily ranks only** (section 8).

### 6.4 Bounty Shop (`bounty-shop`)
- Title "Bounty Shop", back arrow about (222, 75), a hedgehog-like shopkeeper on the left. Currency numbers top right (changing, ignore).
- Row of cards on the right, starting at x about 1065:
  - **Free Bounty** (yellow, with "Claim" at about (1268, 918), and a red dot) when claimable.
  - **Free Bounty** (dark, text "Next free in 18h09m") when already claimed. The text is a countdown: match only the static part "Next free in".
  - **D Bounty** (3000), **C Bounty** (10000), further right a third card and more: all **priced**, with "1 left today".
- Allowed action here: tap **Claim** only when the yellow Claim state is present, then back out. **Never tap any priced card.**

### 6.5 Host lobby (`lobby/host`)
- Back arrow at about (215, 75). Header with boss portrait, boss name and level ("Green Face Lv.40", "Giant Bat Lv.40", "Shikigami - Wicked Lv.40"), a **Rewards** line, and a reward description (for example "Obtain random C-... Relic").
- Slots (3 for C-rank, 5 for A-rank): your slot first, labeled **Leader** (blue tag), then other players. Empty slots show **+ Invite**.
- Bottom right:
  - Checkbox "Allow members to invite friends" (checked by default). **Leave as is.**
  - "Maximum Relay Times:1".
  - Blue **Team Invite** button at about (1680, 1088).
  - Text "The team needs at least 2 players" when nobody has joined.
  - A green **Go** button at about (2060, 1095). **Observed:** the Go button is **absent** while nobody has joined (the text "The team needs at least 2 players" is shown there) and **appears about 0.2 s after a player joins** and shows "Ready" (a toast "<name> joined the team" appears over the slots at the same moment).
- Bottom left: friends/chat/emote icons and a "Battle" avatar. **Ignore all of them** (user instruction: ignore the battle icon). Never tap the emote icons.
- Other players' slots show a status tag: **Ready** (green), **Adjusting...** (purple).

### 6.6 Team Invite popup (`popup/team-invite`)
- Title "Bounty Team Invite", boss name and level, **Minimum Rank** selector (default "Senior Ninja") and **Select Channel** selector (default "Team Recruitment Channel"), and a yellow **Send Message** button at about (1290, 995).
- Leave both selectors on their defaults. Tap **Send Message** only.
- **Cooldown (measured):** after Send Message the button turns **gray** and shows "**Can send messages after 01m**" then "**59s**" and counting down, so there is a **60 s cooldown** between invites. The popup **stays open** after sending, so close it by tapping the empty area to the right of the slots (the recording shows a tap there about 1.5 s later and the popup closes). VERIFY the back button also works. Set `invite_repeat_seconds` to at least 65, and only tap Send Message when the yellow (not gray) state is visible.

### 6.7 Joined lobby (`lobby/joined`, `lobby/joined-ready`)
- Same layout as the host lobby but you are not the Leader (someone else is). Your slot shows your name (A1).
- Bottom right button:
  - Green **Ready** (about (2065, 1098)) means you have not readied yet. Tap it **once**.
  - Gray **Cancel** at the same spot means you are ready. **Never tap Cancel.** Wait for the battle.
- Other slots show Leader / Ready / Adjusting...

### 6.8 Loading (`battle-loading`)
- Black screen with one tall card per player (character art, name) and a load percentage at the bottom of each card (for example 21% and 0%). Smiley/emote icons at the bottom left. Do nothing. Wait. Loading can take a while (the reference log shows about 9 s), so only apply a long timeout.

### 6.9 Battle start banner
- A black ink band across the middle with white text "Battle Start". Do nothing.

### 6.10 Battle HUD (`battle`)
- Side-scrolling stage. Top left: two feather icons (with small numbers). Top center: a timer-like value. Top right: player names and scores. Right edge: a smiley emote button at about (2390, 735), **never tap it**.
- Bottom row: a down arrow (about (490, 1035)), skill buttons with **cooldown numbers** that change every frame (about (835, 1035), (1090, 1020), (1277, 1020), (1742, 1035)), and a dash button (about (2100, 1040)). **The bot does not use them. Ignore the numbers.**
- The **ultimate** is the round icon with an orange ring on the **left middle** at about (240, 560).

### 6.11 Battle logic (user-specified)
1. After the "Battle Start" banner, the bot **does nothing** (normal HUD).
2. A **red WARNING screen** appears: the whole screen tints red with a large "WARNING!!" in the center. It appears **twice back to back**: first a bit smaller, then a bit bigger, then disappears completely.
3. After the warning has **completely** disappeared, tap the **ultimate** (left middle).
4. After that, many wild animations play. **The bot must not react or alert to anything on screen.** It just waits for the results sequence (6.12).
- Daily bounty battles cannot be lost. There is no defeat handling.
- Detection of the warning: use the fraction of strongly red pixels in a downscaled frame (jumps from about 0 to a large share), confirmed by a "WARNING" text template. "Warning over" means **no red for a quiet period longer than the gap between the two pulses** (start at 0.7 s, tune from logs). Log timestamps of every red-on and red-off event so the real gap can be measured.
- Tap the ultimate **once per warning episode** (one episode = both pulses). VERIFY whether the ultimate must be tapped again for later warnings (section 15).
- **Measured in the user's recording (section 18):** the ultimate icon flashes (becomes ready) at 52.0 s, pulse 1 (small WARNING) runs 52.45 to about 53.15 s, pulse 2 (big WARNING) runs 53.22 to 54.28 s, the warning is fully gone at about **54.4 s**. The gap between the pulses is only about **0.05 to 0.1 s**, so a quiet period of 0.7 s is safe.
- **Ultimate timing, measured in two recordings (sections 18 and 18b):** the ultimate was activated **about 3.6 s after the warning disappeared** in the first run (warning gone 54.4 s, ultimate 58.0 s, a B bounty) and **about 3.65 s after** in the second run (warning gone 49.2 s, ultimate 52.85 s, a D bounty with a different boss). The two numbers agree, so the delay looks deliberate (the boss attack lands a few seconds after the warning). The user's own wording was "tap after the warning disappears", so confirm with them (section 15, item 15). Until then make it configurable: `ultimate_delay_after_warning_s`, default **3.6**, and log both the warning-end and tap times. Activation shows as the ultimate icon turning **white** and a skill-name banner (for example "Crimson Feathers Slash") at the bottom left.
- **Do not use "red ratio back to zero" as the warning-over signal.** In the first run (a flaming-sword boss) a large red swirl fills the right side from 54.8 s to 58.0 s and the red ratio rises again. In the second run (a flying red head) red stays near 0.5 %. Detect the warning with the "WARNING" text template plus the full-screen tint (red above about 9 % of the frame), and treat any later red as a different event.
- Robustness idea (proposal, not user-confirmed): after the delay, tap the ultimate, then check for the ultimate cutscene (the icon turns white and a skill name such as "Crimson Feathers Slash" appears bottom left). If it did not start within about 1 s, tap again, at most 3 times. The ultimate button is a single-purpose spot, so a repeat tap is low risk.

### 6.12 Results sequence
**Order confirmed from the recording (section 18):** after the battle ends there is a fade-out, a black gap, a "Crusade successful!" completion card animation (about 1.5 s, nothing to tap), then:
1. **Results screen** appears first ("B Bounty" title with the boss portrait, a row per player with score and rewards, your row highlighted, "MVP" tag possible, yellow **Confirm** button).
2. About **0.8 s later** the **Congratulations popup** opens on top of it (reward icons on a black band, text "Tap the blank space to close"). Tap an empty spot inside the black band, for example about (1288, 900), which is clear of every reward icon. The popup stays until tapped.
3. The Results screen is visible again. Tap the yellow **Confirm** button at about (1303, 1120).
4. Confirm leads to a black **Loading** screen (small "Loading" icon at the bottom right, about 2.6 to 2.8 s). **What comes next depends on the run:** after a **joined** bounty the game shows the **main screen**; after a **hosted** bounty it shows the **Bounty screen** (carousel). So the bot must detect the screen and reopen the chat after joined runs, and move on to the next owned bounty after hosted runs.
- The detector must treat "Results screen alone" and "Results screen plus Congratulations popup" as two different views (`results` and `rewards-popup`), and must not tap Confirm while the popup is open (it is dimmed behind it).
- **Results screen forbidden zones (never tap):** the add-friend icon (blue ninja with plus) and the thumbs-up on each player row (x about 1978 and 2140, one per row), the "Share and Favorite" button at about (365, 1120), and the emote icons at about (510, 1128) and (640, 1130).
- From the results title, read the **rank letter** and increment the local counter for that rank (section 4.3).
- After Confirm the game returns to some earlier screen. VERIFY which one, and let the view detector decide what to do next.

### 6.13 Ignored overlays
- A **friend invite** bar can appear at the top of the screen (an "Invite" text with a countdown, for example 54s). It is a friend inviting the user to their bounty and has **nothing to do with the bot**. **Never tap it or accept it.**
- Other popups, red dots, badges: ignore unless they block a required tap, in which case log them and use the known close path. Unknown blocking popups go through the watchdog (section 9.4).

---

## 7. Flows

### 7.1 `startup`
1. Verify the scrcpy window (position, size, live frames).
2. Identify the current view. Navigate to `main` if the bot knows how, otherwise stop and ask the user to put the game on the main screen.
3. Run `read-limits`, then `claim-free-bounty`.

### 7.2 `read-limits`
Open the Bounty screen, expand the Member Reward Limits panel if collapsed, read each counter (digit templates, OCR fallback), save to `state.json`, collapse the panel if needed. Compute `needed[rank] = cap - done` for each enabled rank.

### 7.3 `claim-free-bounty`
Main, Bounty, Bounty Shop. If the yellow **Claim** state is present, tap it, wait for the confirmation or reward popup (VERIFY what appears), then back out. If the card shows "Next free in", do nothing and back out. **Whitelist rule:** on this screen the only tappable targets are Claim and the back arrow.

### 7.4 `host-bounty` (hosting phase, runs first)
1. On the Bounty screen, **tap a side card** to bring the wanted bounty to the center (wait about 0.5 s for the animation), then tap the **centered** card. Wait for the host lobby (black screen about 0.4 s, lobby about 1.0 s after the tap, allow a grace window).
2. Tap **Team Invite**, then **Send Message** with default settings (the popup appears about 0.2 s after the tap). Close the popup by tapping the empty area to the right of the slots. The invite has a **60 s cooldown**: re-send at most every `invite_repeat_seconds` (65 or more). In the recording the first joiner arrived about **10 s** after the invite.
3. Wait for a joiner. **As soon as the green Go button appears, tap Go immediately** (it appears about 0.2 s after a player joins). Do not wait for additional players (user decision). The recording shows loading starting about 1.8 s after the user's first Go tap, so allow a grace window before retrying.
4. Battle, then results (6.11, 6.12).
5. Repeat with the next owned daily bounty (selection loop in section 8). Stop hosting when no owned daily bounty remains, or `host_wait_timeout_seconds` (config, default 600) passes with no joiner for one bounty (then leave the lobby and move on to the next, VERIFY what leaving does to the bounty).

### 7.5 `join-bounty` (joining phase, runs after hosting)
0. **After every finished bounty the game is on the main screen** (see 6.12). Reopen the chat (chat icon on the main screen, or the chat bubble on the Bounty screen).
1. The chat opens on the general **Chat** tab (which has a text input and a **Send** button, **never tap those**). Tap the **Team Recruitment** tab and **park there**. VERIFY whether the game remembers the last tab.
2. Scan only the chat ROI, newest (bottom) first. For each yellow **Join** button found, pair it with its header strip (238 px above) and read the rank letter and boss name by template.
3. If the rank is **still needed** (counter below cap and rank enabled), re-grab a fresh frame, re-match that card's Join, and tap it immediately.
4. **Verify**: within about 1 s the joined lobby must appear. If not (the card was stale or full, or the list shifted), log a miss and resume scanning.
5. In the joined lobby, tap **Ready** once. Wait. Battle, results, then back to the chat and repeat.
6. Stop when no enabled rank is below its cap, or after `max_join_run_minutes` (config).

### 7.6 `battle`
As in 6.11, then `finish-results` as in 6.12.

---

## 8. Decision logic

- Enabled ranks and caps come from `config.yaml` and from the Reward Limits panel. `needed[rank] > 0` means the bot wants that rank.
- Hosting order (user decision): **no particular order**. Host **every owned bounty whose rank is daily** (D, C, B, A, A+), regardless of the participant cap. Skip owned bounties of weekly ranks (S, S+, SS, SS+) unless `host_ranks` in the config says otherwise.
- Carousel selection loop (user-confirmed carousel rules): the Bounty screen shows **at most 5 owned bounties at once**: one large card in the middle and up to **2 silhouette cards on each side**. The number of owned bounties **differs every day**, so the bot must handle 1 to many (when there are fewer than 5 the side slots are simply empty, and when there are more, extra cards are off screen). **No swiping is ever needed.** To scroll, tap the card **immediately next to the middle one**. Tapping a card two places away does **not** jump two cards, only the adjacent card responds, so to reach a farther card tap the adjacent one repeatedly, one card per tap, and wait about 0.5 s for the animation each time. After each hosted run the carousel re-centers on the first remaining card. Loop: read the **rank letter** on the centered card. If it is a daily rank, tap the centered card to host it. If not, tap the adjacent card on the side where unvisited cards remain, wait for the animation, and read again. Keep a per-run list of cards already visited (rank plus boss name plus level) so the bot never loops forever, and stop hosting when the carousel is empty or every remaining card has been visited or has a skipped rank. Then run `read-limits` again and move on to joining.
- Joining: a chat card is a **candidate** if its rank is needed. If several candidates are visible, prefer the one lowest on screen (newest), then the rank with the higher remaining need.
- A+ counts toward the **A/A+** counter. S and above are weekly counters and are **disabled by default**.
- Never join a card whose minimum rank the player does not meet (VERIFY how this is shown, the cards seen say "Senior Ninja").

---

## 9. Timing, grace windows and watchdog

### 9.1 Debouncing
Change a view only after the new view is seen for **2 consecutive frames** (except the warning logic in 6.11, which has its own quiet period).

### 9.2 Grace window after an action
After a tap, the next screen may be a transition with **no template** (the reference bot logs a short `unknown` between a Join tap and the lobby). Do not call this a failure immediately. Wait up to `grace_ms` (config, default 1500, per action) for the expected view. Only if it never appears, log `action outcome mismatch` and recover.

### 9.3 Timeouts (all in config)
- Loading screen: long (for example 120 s).
- Host lobby waiting for a joiner: `host_wait_timeout_seconds`.
- Battle (from banner to results): `battle_timeout_seconds`, default 300, to be tuned from real fights.

### 9.4 Watchdog
If no known view has been recognized for `watchdog_seconds` (default 180), **stop tapping**, save the last frames, and notify the user (log line, console beep, optional desktop notification). The bot does **not** react or alert to battle animations (section 6.11), only to long periods with no recognizable screen, and the user can raise or disable this.

---

## 10. Safety (mandatory, implement before any real taps)

1. **Kill hotkey**: F12 stops all taps immediately and exits.
2. **Rate cap**: at most `max_taps_per_minute` taps (default 60), a hard ceiling in the input layer.
3. **Per-view whitelist**: the input layer only allows taps on regions that the current view lists as allowed. Everything else is blocked and logged.
4. **Forbidden zones** (always blocked, in every view): priced shop cards, add-friend and thumbs-up icons on the results screen, "Share and Favorite", emote and smiley buttons everywhere, friend-invite bars, the Discord bubble, the Cancel button in a ready lobby, the lobby back arrow (except on an explicit leave path), skill buttons in battle other than the ultimate.
5. **Never buy anything.** No code path may tap a priced card.
6. **Dry-run mode** (default on for the first run of each flow).
7. **Window guard**: stop if the scrcpy window moves, resizes, or loses live frames.
8. **Run summary** at exit: bounties hosted, bounties joined, ranks completed, misses, unknown-screen events, and any stop reason.

---

## 11. Logging, debugging and replay

- Log style (mirrors the reference bot): short event ids, then fields, with repeated identical lines collapsed:
  - `view=lobby/joined confidence=1.000`
  - `action=join-bounty target=post rank=B index=1`
  - `flow_started flow=bounty-recruitment` / `flow_finished flow=... result=...`
  - `action outcome mismatch: ... expected=... observed=unknown/unknown`
  - repeats: `view=battle-loading repeats=7 confidence=0.985..1.000 duration=9.16s`
- Save a frame to `logs/frames/` whenever: the view is `unknown` for more than a second, an action mismatch happens, or the watchdog fires.
- **Replay tool**: `tools/replay.py <folder>` runs view detection on a folder of saved screenshots and prints the result for each. All view detection must be testable offline this way. This is the main debugging and token-saving tool.
- **Template cropping tool**: `tools/crop_template.py` crops a rectangle out of a screenshot at the working scale and saves it into `templates/` with the right name.

---

## 12. Suggested project layout

```
bounty-bot/
  CLAUDE.md
  PROGRESS.md
  config.yaml
  state.json              (generated)
  templates/              (cropped templates, named view__anchor.png)
  screens/                (the user's reference screenshots, for tests)
  logs/                   (logs and saved frames)
  bot/
    capture.py            (scrcpy window capture, downscale)
    vision.py             (matchers, color/red-ratio checks, digit reading)
    views.py              (view definitions and detection)
    actions.py            (taps, whitelist, forbidden zones, rate cap, jitter)
    flows/                (startup, read_limits, claim_free, host, join, battle, results)
    state.py              (counters, state.json)
    safety.py             (kill hotkey, window guard, watchdog)
    main.py
  tools/
    replay.py
    crop_template.py
```

`config.yaml` should include: working frame size, template threshold values, `enabled_ranks`, `host_ranks` (default D, C, B, A, A+), `max_taps_per_minute`, `invite_repeat_seconds`, `host_wait_timeout_seconds`, `battle_timeout_seconds`, `watchdog_seconds`, `grace_ms` per action, `warning_quiet_seconds`, jitter ranges, dry-run flag, window position and size.

---

## 13. Build order (finish and test each step on the real phone)

1. **Capture**: launch scrcpy, grab the window, show live frames and print fps. Success: stable 30+ fps and a stored frame saved on demand.
2. **Templates and replay**: build `crop_template.py` and `replay.py`. Using the user's screenshots in `screens/`, make views `main`, `chats/recruitment-chat`, `bounty-owned`, `bounty-shop`, `lobby/host`, `lobby/joined`, `lobby/joined-ready`, `battle-loading`, `battle`, `rewards-popup`, `results` detectable. Success: replay labels every saved screenshot correctly.
3. **Input layer with safety**: taps through the scrcpy window with whitelist, forbidden zones, rate cap, kill hotkey, dry-run. Success: dry-run logs the intended tap and a live tap hits the right button once.
4. **Navigation**: main to Bounty to Bounty Shop and back, claim-free-bounty. Success: claims the free bounty when available, does nothing otherwise.
5. **Read limits**: open the panel, read counters, update `state.json`. Success: matches what is on screen.
6. **Battle and results**: warning detection (log the gap between the two pulses), ultimate tap, rewards popup, results Confirm, counter update. Success: one full battle from banner to Confirm without human input.
7. **Hosting flow**: carousel tap, host lobby, invite, wait, Go when green, battle, results.
8. **Joining flow**: chat scan, rank pairing, fast Join tap with verification, Ready, battle, results.
9. **Orchestrator**: startup, read limits, claim, host, join, stop conditions, run summary, watchdog.
10. Tuning from logs: thresholds, quiet period, grace windows, timeouts.

---

## 14. Template and capture checklist

Needed from real frames (the user has already provided one example of most of these screens):
- Main screen lettering anchors (Bounty, Story, Race Mode, Arena, Clan) and the Chat icon.
- Chat: tab labels, the yellow Join button, the "Expired" text, the rank-letter glyphs in the card header (D, C, B, A, S, and still needed: **A+, S+, SS, SS+**), and each boss name or portrait.
- Bounty screen: title, centered-card frame, the Member Reward Limits label with both chevron states, the digit glyphs for counters.
- Bounty Shop: title, yellow "Claim" state, "Next free in" state.
- Host lobby: Team Invite button, green Go button, "+ Invite" slot, Leader tag, Ready tag, "Adjusting..." tag.
- Team Invite popup: title and Send Message button.
- Joined lobby: green Ready button, gray Cancel button.
- Loading: the black screen with player cards and percentages.
- Battle: HUD anchors (feather icons, top-right player panel), the "Battle Start" banner, the red WARNING screen (text and red ratio reference), the ultimate icon.
- Results: "Congratulations" popup, results title (with rank letters), Confirm button.

---

## 15. Open questions (do not guess, ask the user or capture a screenshot)

1. ~~Which bounty to host first, and how the carousel scrolls~~ **Answered:** no particular order, all owned **daily** bounties, weekly-rank ones skipped. The carousel shows at most 5 cards (middle plus 2 each side), the count varies daily, no swiping, only the card **adjacent** to the middle can be tapped to scroll by one.
2. What happens to a hosted bounty if the lobby is left without a battle (is it consumed?), and how to leave the lobby.
3. ~~Invite cooldown~~ **Answered:** 60 s cooldown, popup stays open after sending. Still open: whether the chat message actually posted (confirm in the recruitment chat).
4. Whether the ultimate must be tapped again on later warnings, what the orange ring means (readiness?), and whether the ultimate has a cooldown.
5. The real time gap between the two warning pulses (measure with logs), and whether to tap right after the quiet period or after a short extra delay.
6. When the daily counters reset (time of day), and how the bot should detect the reset.
7. Whether weekly S, S+, SS, SS+ bounties should be enabled.
8. How A+, S+, SS, SS+ look in the chat header, and which bosses go with which ranks.
9. ~~Which screen the game returns to after Confirm~~ **Answered by the recording:** Loading screen, then the main screen.
10. ~~Go button before a joiner~~ **Answered:** it does not exist until a joiner is Ready (the text "The team needs at least 2 players" is shown instead).
11. The funnel (filter) icon popup in the chat, whether it can filter by bounty type, and whether the bot should use it.
12. How a player below the "Minimum Rank" on a card is blocked from joining.
13. How to close the chat panel.
14. What the Free Bounty claim shows after tapping Claim.
15. **Ultimate timing:** both recordings show the ultimate activated about **3.6 to 3.65 s after the warning disappears**, with two different bosses. The user's wording was "after the warning disappears". Confirm that the delay is intentional (and whether any time after the warning is acceptable), and whether the ultimate may be fired earlier (its icon is ready before the warning).
16. The joined-lobby wait: in the recording about 15 s passed between pressing Ready and the leader starting. Should the bot leave if the leader never starts (after how long)?

---

## 16. Reference notes: another player's bot (for context only)

A friend of the user built a similar bot, described in a chat (no code available, the author did not even push it to a repository):
- Identifies the current screen by comparing screenshots with **template crops** (buttons and similar) using algorithms like **brightness difference**. Described as extremely fast.
- **OCR** is used only in limited cases, for example double-checking bounty progress.
- Handles daily bounties of rank D, C, B, A, A+, and potentially S, S+, SS. It **cannot fight bosses**.
- Navigation latency about **10 to 100 ms**. He once optimized it so far it "reacted too fast" and reverted that change.
- Takes about 20 minutes of the user's day plus the social part (chat). Can reset or buy bounties from the store (this project **does not buy**).
- Works from any video stream. His setup is a PC connected to an Android phone by cable, no emulator. BlueStacks was researched and easy to connect, and screen sharing with taps allowed (for example AnyDesk) could work.
- Built with an AI coding agent over roughly a month, mostly to make it work **every time**, not just once.
- His log vocabulary (views with confidence, actions, flows, `action outcome mismatch`, repeated-line collapsing) is the model for section 4 and section 11.

---

## 18. Measured timeline from the user's screen recording

Source: `Screen_Recording_20261031_115202_NinjaMustDie.mp4` (1560 x 720, 60 fps, 79.9 s, same aspect ratio as the 2576 x 1189 screenshots). It is one **joined** B bounty ("Crimson Flame Sword Lv.60", event tag "Dragon Contract Boosting", 3 players, leader "Tak", the user is A1). The phone shows **touch dots** (show-touches is on), so tap moments are visible. Times below are video seconds. Put the video in `recordings/` and re-derive these numbers with the frame tools (section 19) before trusting them for tuning.

| Time (s) | Event |
|---|---|
| 0.0 - 1.7 | Chat panel open on the general **Chat** tab (messages, a text input and a Send button) |
| 1.7 - 1.8 | Tap on the **Team Recruitment** tab, tab switches within about 0.1 s |
| 1.8 - 2.9 | Recruitment cards visible (C Bounty "Thunder Dragon" 1/3, B Bounty "Crimson Flame Sword" 1/3), each with a yellow Join. The Discord bubble slides in |
| **2.9** | Tap on the **Join** button of the B card (first touch dot) |
| **3.5** | Joined lobby is on screen with a green **Ready** button: about **0.5 to 0.6 s** from the first touch dot to the lobby (a short `unknown`/transition in between) |
| **4.2** | Tap on **Ready**, the button becomes gray **Cancel** and your slot shows "Ready" by **4.4** (about 0.2 s) |
| 9.0 - 12.0 | A player left, the middle slot shows "+ Invite" |
| 12.5 - 14.5 | A toast "(player) joined the team" appears over the slots, a new player fills the slot. **Lobby contents change while you wait, do not match on character art or names** |
| ~19.0 | All players Ready, the leader starts the battle |
| 19.7 - 39.6 | Black **loading** screen with one card per player and a load percentage (about **19.9 s**). One player stayed at 0% until about 27 s, so the slowest player decides |
| 39.5 - 44.4 | Stage fades in, characters run. HUD not yet visible |
| 44.5 - 45.7 | **"Battle Start"** banner |
| 46.5 | Skill buttons appear along the bottom |
| 52.0 | Ultimate icon (left middle) flashes, it is ready |
| 52.45 - 53.15 | **WARNING pulse 1** (smaller text) |
| 53.22 - 54.28 | **WARNING pulse 2** (bigger text), warning fully gone at about 54.4. Gap between the pulses 0.05 to 0.1 s |
| 54.8 - 58.0 | Large red swirl and flaming sword attack on the right side (a second red phase) |
| **~58.0** | **Ultimate activated** (cutscene "Crimson Feathers Slash"), about **3.6 s after the warning disappeared** |
| 58 - 68 | Wild animations and effects (no action needed) |
| 67.9 - 69.6 | Black screen |
| 69.6 - 70.9 | "Crusade successful!" completion card animation |
| **71.1** | **Results screen** appears |
| 71.8 | **Congratulations popup** opens over it (about 0.8 s later) |
| ~73.1 | Popup closed by a tap, results screen visible again |
| **75.5** | Tap on **Confirm** |
| 75.9 - 78.7 | Black **Loading** screen (icon bottom right), about 2.8 s |
| 78.7 | **Main screen** |

### 18b. Second recording: a **hosted** D bounty

Source: `Screen_Recording_20261031_121022_NinjaMustDie.mp4` (1560 x 720, 60 fps, 71.8 s). The user hosts "Red Face Lv.30" (D bounty), one joiner ("Crede"). Times in video seconds.

| Time (s) | Event |
|---|---|
| 0.0 - 1.0 | Main screen, tap on the Bounty tile, Bounty screen opens (carousel) |
| 2.9 | Tap on the **side card** (Red Face), it moves to the center by about 3.15 |
| 4.15 | Tap on the right side card (C Thunder Dragon), it centers by 4.4 (the user is browsing) |
| 5.65 | Tap on the left side card (Red Face) again, centered by 6.15 |
| ~7.15 | Tap on the **centered** card |
| 7.45 - 7.95 | Black screen |
| **8.2** | **Host lobby**: you are Leader, two "+ Invite" slots, blue **Team Invite**, text "The team needs at least 2 players" (about 1.0 s after the tap) |
| ~9.3 | Tap on **Team Invite**, popup "Bounty Team Invite" visible at 9.45 |
| **~10.7** | Tap on **Send Message**, the button turns gray at once: "Can send messages after 01m", then 59s (60 s cooldown) |
| ~12.2 - 12.45 | Tap on the empty area right of the slots, popup closed at about 12.45 |
| **21.1 - 21.2** | A joiner appears, toast "Crede joined the team", slot shows **Ready**, green **Go** button appears (about 10 s after the invite) |
| ~22.6 | First tap on **Go** (touch ring stays through 24.2, the user may have tapped more than once) |
| 24.4 - 34.4 | Black loading screen with two player cards and percentages (about **10 s**, two players) |
| 34 - 38.9 | Stage fades in, characters run |
| 39.0 - 40.4 | **"Battle Start"** banner |
| 41 | Skill buttons appear |
| 47.25 - 47.9 | **WARNING pulse 1** (small) |
| 48.0 - 49.15 | **WARNING pulse 2** (big), warning gone at **49.2** |
| 50 - 52.8 | A flying red head (the boss) approaches from the right |
| **~52.85** | **Ultimate activated** (icon turns white, "Crimson Feathers Slash"), **about 3.65 s after the warning disappeared** |
| 53 - 60 | Effects |
| 60.8 - 62.0 | Black screen |
| 62.0 - 63.4 | "Crusade successful!" completion card |
| **63.65** | **Results screen** appears |
| **64.15** | **Congratulations popup** opens (about 0.5 s later), closed by a tap at about 65.5 |
| **~67.65** | Tap on **Confirm** |
| 68.0 - 70.6 | Black **Loading** screen |
| **70.6** | **Bounty screen** (carousel) with the first remaining bounty centered, the hosted one is gone |

Notes: a hosted run with one joiner took about **60 s** from the first tap on the card to the Bounty screen, and about **26.5 s** from the Battle Start banner to the results screen (similar to the joined run, 26.6 s). Loading is shorter with two players (10 s) than with three (20 s).

Derived numbers for config defaults: lobby (all Ready) to loading about 0.7 s; loading about 20 s; banner to results about **26.6 s** for this boss; battle start (banner) to ultimate about **13.5 s**; whole join run from Join tap to main screen about **76 s** (a good part of it waiting for other players). Use these as starting values for the timeouts in section 9.3 (for example `loading_timeout_s` 90, `battle_timeout_s` 180, `results_timeout_s` 30).

Detection hints from the video:
- Loading screen: more than 80% near-black pixels plus player cards. The post-Confirm Loading screen is also mostly black but has **no cards** and a tiny icon at the bottom right, so tell them apart by the cards.
- The ultimate icon has visible states: ready (orange ring around a red icon), activated (white icon). Use it as a readiness signal.
- Do not use the overall red ratio on its own for the warning (see 6.11).

---

## 19. Working with recordings

Recordings are a first-class source of truth for timing and transitions:
- Record with scrcpy so the capture matches what the bot sees: `scrcpy --no-audio --record=run.mp4 --show-touches`.
- Extract frames with `ffmpeg`, for example a timestamped contact sheet: `ffmpeg -i run.mp4 -vf "fps=2,scale=390:-1,drawtext=text='%{pts\:hms}':x=6:y=6:fontsize=22:fontcolor=yellow:box=1:boxcolor=black@0.6,tile=4x8" -frames:v 5 sheet_%02d.png`.
- Work in two passes: a low-rate overview first, then dense frames (10 to 60 fps) only around the moments that matter. Never decode every frame into context.
- Measure with code, not by eye: per-frame red ratio, dark ratio and ROI template scores over time. `tools/redseries.py` style scripts are fine.
- Save the extracted key frames into `screens/` and use them as test cases for `tools/replay.py`.

---

## 20. Risk note

Automating a game can violate its terms of service, and accounts can be restricted. The user has decided to proceed. The safety rules in section 10, human-like jitter in 3.3, and conservative rate limits reduce, but do not remove, that risk. Do not add features that go beyond the scope in section 1 (no buying, no friend requests, no chat messages beyond the standard bounty invite).