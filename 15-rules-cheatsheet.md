# 15. The rules cheatsheet

Every rule from the guide that tends to surprise people, in one list, grouped by subsystem. Each one links to the full explanation.

## Loading

1. `PUPFolder` points to the folder **containing** `pupvideos`. A pack next to the table wins over the global folder. → [chapter 1](01-concepts-and-setup.md#how-vpx-finds-a-pack)
2. `Init` then `B2SInit` (or PinMAME) for the same game: `screens.pup` and `playlists.pup` are **never loaded**. → [chapter 1](01-concepts-and-setup.md#c-from-the-script-with-init-only)
3. `Init` on an existing screen **replaces** it with a plain Show screen. → [chapter 8](08-script-api.md#init)
4. `D0` = 1 fires once at pack start: use it to start backgrounds. → [chapter 1](01-concepts-and-setup.md#what-a-full-load-does-in-order)
5. No FFmpeg libraries means no PUP screens at all, labels included. → [chapter 1](01-concepts-and-setup.md#where-screens-appear-windows)
6. `.bat`, `.exe` and `pupinit.bat` are never run. Copy option `.pup` files by hand. → [chapter 2](02-pack-files.md#things-vpx-ignores)

## `.pup` files

7. The first line is always skipped. → [chapter 2](02-pack-files.md#the-pup-file-format)
8. Minimum column counts: screens 6, playlists 7, triggers 13. Shorter lines are skipped and logged. → [chapter 2](02-pack-files.md#the-pup-file-format)
9. No spaces around commas: `" ForceBack"` is an invalid mode, which makes the screen Off. → [chapter 2](02-pack-files.md#the-pup-file-format)
10. Double quotes group fields and are removed. `""` doesn't produce a quote. → [chapter 2](02-pack-files.md#the-pup-file-format)
11. Play file and caption paths accept both `\` and `/`. → [chapter 2](02-pack-files.md#paths-and-separators)

## Screens

12. Only screens 0, 2 (6), 5 (1) have windows. Everything else must be placed with CustomPos. → [chapter 3](03-screens-and-layout.md#screen-numbers)
13. `screens.pup` column 6 "Priority" is a **volume multiplier**, not a priority. → [chapter 3](03-screens-and-layout.md#screenspup)
14. `screens.pup` column 4 "Transparent" does nothing. → [chapter 3](03-screens-and-layout.md#screenspup)
15. The `screens.pup` background plays at volume 0 → 1 %. Use a `D0` SetBG trigger for an audible background. → [chapter 3](03-screens-and-layout.md#the-default-background-columns-2-and-3)
16. CustomPos percentages are of the **window**, even for children of children. Define parents before children for draw order. → [chapter 3](03-screens-and-layout.md#custompos-placing-a-screen-inside-another)
17. CustomPos rotation, aspect pairs and sources 99/100/101 aren't supported. → [chapter 3](03-screens-and-layout.md#custompos-placing-a-screen-inside-another)
18. FN 15 moves a screen but never reparents it. → [chapter 3](03-screens-and-layout.md#custompos-placing-a-screen-inside-another)
19. ForcePopBack children draw under the parent's labels. ForcePop children draw over everything. → [chapter 3](03-screens-and-layout.md#draw-order)
20. `Hide`/`Show` only hide and show the screen: its media keeps playing and its mode doesn't change. → [chapter 3](03-screens-and-layout.md#changing-modes-at-runtime)
21. `Off` screens play nothing, background included. MusicOnly screens are heard, not drawn. → [chapter 3](03-screens-and-layout.md#modes)

## Playlists

22. `AlphaSort` 1 shuffles. 2 doesn't. `playlistadd` shuffles on any non-zero value. → [chapter 4](04-playlists.md#playlistspup)
23. Empty `RestSeconds` = **30 s**. → [chapter 4](04-playlists.md#playlistspup)
24. The `ScreenNum` column is ignored. Every screen has its **own copy** (own rotation, own rest timer). → [chapter 4](04-playlists.md#each-screen-has-its-own-copy)
25. Only media files join the rotation. Hidden files (`.DS_Store`) and others are skipped, but can still be played by name. → [chapter 4](04-playlists.md#which-files-are-in-the-rotation)
26. Sort is byte order: uppercase before lowercase, `10` before `2`. Shuffle happens once per load. → [chapter 4](04-playlists.md#which-files-are-in-the-rotation)
27. PuPOverlays/PuPAlphas images must be png/jpg/jpeg/bmp/gif/apng, or the file name is drawn as text. → [chapter 4](04-playlists.md#special-playlists)
28. PuPFrames is drawn **under** the video. → [chapter 4](04-playlists.md#special-playlists)

## Triggers

29. Trigger letters are case-sensitive. Mechs are `N`, not `M`. → [chapter 5](05-triggers.md#events)
30. `=0` conditions fire at startup for everything that's off. → [chapter 5](05-triggers.md#events)
31. No edge detection: every matching event fires. `B2SData` fires every call. → [chapter 5](05-triggers.md#matching-rules)
32. Rows with **identical** Trigger text on a screen fire together. → [chapter 5](05-triggers.md#matching-rules)
33. `D` values reset after firing. Other letters keep their values. → [chapter 5](05-triggers.md#matching-rules)
34. Negative RestSeconds is a delay: −1…−98 seconds, ≤ −99 **milliseconds**. Firings during the delay are dropped. → [chapter 5](05-triggers.md#what-happens-when-a-row-fires)
35. `Counter N` acts on hits 1, N+1, 2N+1… → [chapter 5](05-triggers.md#what-happens-when-a-row-fires)
36. Empty Volume column = playlist volume. → [chapter 5](05-triggers.md#columns)
37. Empty PlayList = comment row (except PlaySSF and CustomFunc). → [chapter 5](05-triggers.md#columns)
38. Spelling: `SplashResume` (or `SplashReturn`), `SkipSamePrty`. Unknown keywords mean Normal. → [chapter 5](05-triggers.md#actions-loop-column)
39. `Defaults 1` turns any row (even StopPlayer) into "play next file at default volume". → [chapter 5](05-triggers.md#defaults--1-simple-play)
40. PlaySSF needs `"x,y,z"` in Counter and the file name with `.wav`. → [chapter 5](05-triggers.md#playssf)
41. DMD triggers need `PupCapture/<n>.bmp` or an active Serum colorization. → [chapter 5](05-triggers.md#dmd-frame-triggers)

## Playback

42. Equal priority **replaces**. Lower priority is **refused**, not queued. → [chapter 6](06-playback-engine.md#priorities)
43. The background has no priority and doesn't count as "playing". → [chapter 6](06-playback-engine.md#a-screens-state)
44. The playlist rest only applies to Normal plays that don't preempt. Any positive priority on an idle screen skips it. → [chapter 6](06-playback-engine.md#the-rest-gate)
45. Requesting the file that's already playing doesn't restart it. → [chapter 6](06-playback-engine.md#evaluation-order)
46. Priority 999 = stop what is playing (a background-only screen is left alone). → [chapter 6](06-playback-engine.md#stopping)
47. Queued items play **newest first**. → [chapter 6](06-playback-engine.md#queue-actions)
48. Nothing queued behind a looping media ever plays. → [chapter 6](06-playback-engine.md#loop)
49. SplashResume continues the interrupted media or background where it was. SplashReset restarts it. → [chapter 6](06-playback-engine.md#splashreset-and-splashresume)
50. Backgrounds restart from the beginning, except after a SplashResume, which continues them. → [chapter 6](06-playback-engine.md#the-background)
51. `SetBackGround n,1` copies the current media into the background slot (it plays on, then loops). `,0` always clears the slot. Pop and MusicOnly screens never keep it. → [chapter 6](06-playback-engine.md#the-background)
52. Every stop, queue advance and loop pass is an end of media: it resolves hidehudplay, returnplay, FN 34 and ducks. → [chapter 6](06-playback-engine.md#when-media-ends)
53. `SetLength n,s` ends what's playing `s` seconds from now (0 = now), background included. A request's Length counts from the request and belongs to that media only. → [chapter 6](06-playback-engine.md#length-loop-and-pause-from-the-script)
54. png/jpg/bmp from a normal playlist is a static image, shown **only when no video plays**, so it's invisible over a background loop. Stops don't remove it; an accepted video request does. → [chapter 6](06-playback-engine.md#images-in-a-normal-playlist)
55. Refused plays on ForcePop/ForcePopBack screens still reorder them. → [chapter 6](06-playback-engine.md#evaluation-order)

## Volume and sound

56. Volume 0 = 1 %, never mute. Output never exceeds 100 %. → [chapter 7](07-volume-ducking-sounds.md#the-volume-model)
57. `-NN` file names duck the other playing screens. `-name` without digits needs FN 54. → [chapter 7](07-volume-ducking-sounds.md#ducking)
58. Only one duck source at a time. `ALL:1` never auto-restores. A looping `-NN` file releases and re-ducks at each pass. → [chapter 7](07-volume-ducking-sounds.md#ducking)
59. Sounds are only `.wav` files in `PUPSOUNDS`, named with `.wav`. SoundAdd/SoundPlay are ignored. → [chapter 7](07-volume-ducking-sounds.md#sound-effects-pupsounds)
60. Sound position: only x (−10…10) is used. `MainVol` doesn't affect sound effects. → [chapter 7](07-volume-ducking-sounds.md#playing)

## SendMSG

61. Only `mt` 301 and 402 do anything. → [chapter 9](09-sendmsg-custom-functions.md#message-format)
62. `SN 99` = every screen 0–19. → [chapter 9](09-sendmsg-custom-functions.md#message-format)
63. FN 34 waits for the next end of media (including a stop or a background loop pass). Send it after starting the media. → [chapter 9](09-sendmsg-custom-functions.md#supported)

## Labels

64. Fonts must be in `FONTS` (or be Liberation Sans). System fonts are never used, and a missing font means no text. → [chapter 10](10-labels.md#fonts)
65. Same-family font files: use "Family Style" or the file name. → [chapter 10](10-labels.md#fonts)
66. Duplicate `LabelNew` names are rejected, and the first one wins. → [chapter 10](10-labels.md#creating-labels-labelnew)
67. Position 0 = the natural spot for the alignment (50 % for centered) unless `v2`. → [chapter 10](10-labels.md#positioning)
68. `LabelSet` visibility only applies on the current page, unless the caption is `` `u` ``. → [chapter 10](10-labels.md#setting-labels-labelset)
69. Visibility can't change while animating. It's restored to its pre-animation value. → [chapter 10](10-labels.md#setting-labels-labelset)
70. An image label never turns back into text. An empty caption on an image is ignored. → [chapter 10](10-labels.md#image-captions)
71. A missing image clears the label, and hides it if it was showing an image (the "clear.png" trick). → [chapter 10](10-labels.md#image-captions)
72. `LabelShowPage` duration: < 501 seconds, ≥ 501 milliseconds. `500` = 500 s. → [chapter 10](10-labels.md#labelshowpage)
73. Switching to a page > 1 resets its labels to their **created** visibility (`dodefault`). → [chapter 10](10-labels.md#labelshowpage)
74. `forceshow`/`forcehide` are case-sensitive. `hidehudplay`/`returnplay` aren't. → [chapter 10](10-labels.md#labelshowpage)
75. Labels are drawn only with a non-empty caption. → [chapter 10](10-labels.md#pages)
76. Bold isn't rendered. → [chapter 10](10-labels.md#rendering-details)

## Label properties and animations

77. Keys in one special are applied in a **fixed order**, not as written. → [chapter 11](11-label-properties.md)
78. `size` (points) is ~1.5× bigger than `fonth`/LabelNew (cell height) for the same number. → [chapter 11](11-label-properties.md#position-and-size)
79. `width`/`height` only apply to image labels. Images use native size until set. → [chapter 11](11-label-properties.md#position-and-size)
80. Shadows need a shadow type. Offsets alone do nothing. Default 3 px grey. → [chapter 11](11-label-properties.md#shadow-and-outline)
81. `zback:-1` goes below pBackground. Any other value goes just above it. → [chapter 11](11-label-properties.md#order)
82. `dispose` deletes the label for good. → [chapter 11](11-label-properties.md#visibility-and-lifetime)
83. Animations restore visibility/alpha/zoom/color/filter at the end. The visible you pass to `LabelSet` is the end state, **unless the label was already animating**, in which case the old animation's start state wins. → [chapter 12](12-label-animations.md#rules-common-to-every-animation)
84. Move positions `0`, `1`, `-1`, `2` are codes, not percentages. → [chapter 12](12-label-animations.md#at2-move)
85. Wiggle angles are whole degrees. `angle`/`rotate` are tenths. → [chapter 12](12-label-animations.md#at8-wiggle)
86. `at:7` leaves the whole overlay at its end alpha, or at its mid-fade alpha if interrupted. → [chapter 12](12-label-animations.md#at7-overlay-fade)
87. Page switches stop every animation on the screen. → [chapter 12](12-label-animations.md#rules-common-to-every-animation)

## More rules

88. `MainVol`/FN 26 changes don't re-level queued items or later background restarts. → [chapter 7](07-volume-ducking-sounds.md#the-volume-model)
89. A `hidehudplay` splash that ends by timer or by another page change leaves pBackground **hidden**. → [chapter 10](10-labels.md#labelshowpage)
90. `LabelNew` size and positions are whole numbers. Use `fonth`/`xpos`/`ypos` for decimals. Label calls on an undefined screen do nothing. → [chapter 10](10-labels.md#creating-labels-labelnew)
91. Animations replace the label's own alpha and zoom while they run. → [chapter 12](12-label-animations.md#rules-common-to-every-animation)
92. Leaving out `len` makes zoom and count-up animations jump to their end value. → [chapter 12](12-label-animations.md#common-keys)
93. `pDMDLabelClone` positions come out in the top-left corner (percent sent as pixels). → [chapter 13](13-pupdmd-framework.md#animations)
94. FN 34 ending turns FN 3 back on. → [chapter 9](09-sendmsg-custom-functions.md#supported)
95. Capture IDs are the digits right before `.bmp`: name captures `<n>.bmp`. Only 128×32 captures match. A magenta outline limits the match to its inside. Lower IDs are tried first. → [chapter 5](05-triggers.md#dmd-frame-triggers)
96. The same capture never fires twice in a row: another capture must match in between. `ExactColorMatch.txt` is ignored. → [chapter 5](05-triggers.md#dmd-frame-triggers)
