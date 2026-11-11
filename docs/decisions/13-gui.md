# GUI

Settled decisions for this area. Cite them by ID in specs, design notes and commit messages. To change one, follow the process in [README.md](README.md).

| ID | Question | Options considered | Decision | Decided by |
| --- | --- | --- | --- | --- |
| W1 | Where does the compositor run? | In the kernel · As a userspace server | As a userspace server. | Claude's pick, approved |
| W2 | How do apps talk to it? | Our own protocol · Wayland-compatible · X11-compatible | Our own small protocol over local sockets. Apps draw into shared-memory buffers and the compositor assembles them. | Claude's pick, approved |
| W3 | Who draws title bars and borders? | The compositor · Each app | The compositor. The look stays consistent, and a frozen app can still be moved and closed. | Claude's pick, approved |
| W4 | How are frames paced? | As fast as possible · Synced to display refresh with frame callbacks | Synced to refresh where the display reports it, a 60 Hz timer otherwise. Apps draw only when asked. | Claude's pick, approved |
| W5 | How does the compositor avoid redrawing everything? | Full redraw · Damage tracking | Damage tracking from the first version. | Claude's pick, approved |
| W6 | Which visual effects? | Transparency · Shadows · Rounded corners · Animations · Blur | Transparency, shadows, rounded corners and animations. Blur arrives with GPU acceleration after 1.0. | Graeson |
| W7 | What pixel format? | 32-bit with premultiplied alpha, sRGB · Others | 32-bit premultiplied alpha, sRGB. No HDR or color management. | Claude's pick, approved |
| W8 | How are high-density screens handled? | Ignored · Integer scale factors · Fractional scaling | Fractional scale factors from day one. The toolkit works in logical units and the vector rasterizer draws at the device scale. | Graeson, detailed by Claude |
| W9 | Multiple monitors? | One only · Designed for several, one tested | Designed for several, one tested until phase 6. | Claude's pick, approved |
| W10 | What language is the toolkit? | C · C++ | C, with an object-style widget API. C++ would need its runtime ported first. | Claude's pick, approved |
| W11 | What toolkit style? | Retained widget tree · Immediate mode | Retained widget tree. | Claude's pick, approved |
| W12 | How does layout work? | Fixed positions · Box and grid containers · Constraint solver | Box and grid containers. | Claude's pick, approved |
| W13 | Where does 2D drawing come from? | Rectangles and blits only · Our own antialiased vector rasterizer | Our own rasterizer: antialiased paths, rounded rectangles, gradients, image scaling. | Claude's pick, approved |
| W14 | How is text rendered? | Bitmap fonts · Our own TrueType rasterizer · Ported FreeType | Our own TrueType rasterizer with antialiasing and subpixel positioning. Fonts themselves are open-licensed data. | Claude's pick, approved |
| W15 | Which scripts and text features? | Latin only · Latin, Greek, Cyrillic with kerning · Full shaping, right-to-left, emoji | Latin, Greek and Cyrillic with kerning in phase 5. Complex scripts and emoji later, through a ported shaping library. | Claude's pick, approved |
| W16 | Which image formats? | PNG · JPEG · A vector format for icons | PNG and a vector icon format with our own decoders. JPEG later. | Claude's pick, approved |
| W17 | What should it look like? | Retro · Modern flat · Close to an existing desktop · Something of its own | Modern flat, in the spirit of KDE Plasma, with original assets. A one-page design system (spacing, type scale, colors, corner radius) is written before any widget code. | Graeson, detailed by Claude |
| W18 | Light and dark themes? | One theme · Both, from one token table | Both, from one token table, plus an accent color. | Claude's pick, approved |
| W19 | Which window management features? | Snap to edges · Keyboard tiling · Window switcher · Virtual desktops | Snap, keyboard tiling and a switcher in phase 5. Virtual desktops after. | Claude's pick, approved |
| W20 | What makes up the desktop shell? | Panel with launcher, window list and clock · Notifications · Lock screen · Login screen | Panel and login screen in phase 5. Notifications and lock screen next. | Claude's pick, approved |
| W21 | Which apps ship first? | Terminal · File manager · Text editor · Settings · System monitor · Image viewer · Calculator | The first four in phase 5. The rest follow, and each one tests the toolkit. | Claude's pick, approved |
| W22 | How do keyboard layouts work? | US only · Layout files with dead keys · Input methods for CJK | Layout files with dead keys, US plus one other to prove it. Input methods are out of scope. | Claude's pick, approved |
| W23 | How does the pointer feel? | Raw movement · Acceleration curve and smooth scrolling · Touchpad gestures | Acceleration curve and smooth scrolling. Gestures later. | Claude's pick, approved |
| W24 | Clipboard and drag-and-drop? | Clipboard · Both | Typed clipboard in phase 5. Drag-and-drop after. | Claude's pick, approved |
| W25 | What accessibility support? | None · Full keyboard navigation and scalable text · Screen reader | Keyboard navigation and scalable text from the start. Widgets expose roles and labels so a screen reader stays possible. | Claude's pick, approved |
| W26 | How are apps packaged and launched? | Loose binaries · One directory per app with a manifest | One directory per app with a manifest: name, icon, executable, file types it opens. | Claude's pick, approved |
| W27 | What responsiveness rules apply? | None · Written rules | Two written rules: an app's UI thread never blocks on I/O, and the compositor never waits on an app. | Claude's pick, approved |
| W28 | What happens when an app crashes? | Could take the desktop down · Isolated | Isolated. Compositor restart with apps reconnecting is a stretch goal. | Claude's pick, approved |
| W29 | Can the screen be captured? | No · Screenshot call | Screenshot call in phase 5. Automated GUI tests compare frames with it. | Claude's pick, approved |
| W30 | Are there interface guidelines for apps? | None · A short written guide | A short written guide, so every app agrees on menus, shortcuts and dialogs. | Claude's pick, approved |
