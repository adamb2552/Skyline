# Setup before starting in Claude Code

## 1. Hardware and Xcode
- A Mac (Apple Silicon recommended). iOS apps can only be built with Xcode on macOS.
- Install the latest **Xcode** from the Mac App Store, open it once to accept the license and install the iOS Simulator runtime.
- Run `xcode-select --install` for the command line tools.
- Install Homebrew (brew.sh), then: `brew install xcodegen swiftlint swiftformat`

## 2. Apple account
- **To run on your own iPhone:** sign into Xcode with your free Apple ID (Xcode → Settings → Accounts). Free builds expire after 7 days.
- **To use TestFlight or the App Store:** join the Apple Developer Program ($99/year) at developer.apple.com. Not needed until milestone M12, but enrollment can take a day or two.
- On the iPhone, turn on Developer Mode (Settings → Privacy & Security) the first time you run a build from Xcode.

## 3. GitHub
- Create a GitHub account if needed and a new **private** repo (e.g. `skyline`).
- Clone it to your Mac.
- Copy `DESIGN.md` into `docs/DESIGN.md`, and `CLAUDE.md` into the repo root. Commit and push.

## 4. Claude Code
- You need a Claude Pro or Max plan (or an API key).
- Install: `curl -fsSL https://claude.ai/install.sh | bash` (or see docs.claude.com for other options).
- In Terminal: `cd ~/path/to/skyline` and run `claude`. Sign in when prompted.
- Run Claude Code **locally on the Mac** for this project. Cloud sessions run on Linux and can't build the iOS app or run the Simulator (they can still run `swift test` on the simulation package).

## 5. First prompt
> Read CLAUDE.md and docs/DESIGN.md. Then build milestone M0. When done, tell me how to run it in the Simulator and what to check.

Then go milestone by milestone, e.g. "Build M1 from the design doc. Run the tests before finishing."

## Tips
- Ask Claude to keep `docs/PROGRESS.md` updated so a new session can pick up where the last one stopped.
- Use Plan mode (Shift+Tab) for the bigger milestones (M4 routing, M5 elevators) so you can approve the approach first.
- If something feels off in play, describe what you saw and the steps to reproduce it; Claude can turn that into a `sim-cli` scenario or test.
