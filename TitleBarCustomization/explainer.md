# Title Bar Customization for Web Apps

## Table of Contents
 - [Introduction](#introduction)
 - [Examples of title bar customization on desktop apps](#examples-of-title-bar-customization-on-desktop-apps)
 - [Problem to solve: PWA Title bar area is system reserved](#problem-to-solve-pwa-title-bar-area-is-system-reserved)
 - [Goals](#goals)
 - [Proposal](#proposal)
   - [Overlaying Caption Controls](#overlaying-caption-controls)
   - [Working Around the Caption Control Overlay](#working-around-the-caption-control-overlay)
   - [Defining Draggable Regions in Web Content](#defining-draggable-regions-in-web-content)
 - [Example](#example)
 - [Privacy Considerations](#privacy-considerations)
 - [Open Questions](#open-questions)

## Introduction 

PWAs hosted within a user agent (UA) frame are able to declare which browser display mode best meets the needs of the application via the manifest file's [`display` member](https://developer.mozilla.org/en-US/docs/Web/Manifest/display). Currently, there are 4 supported values and their behaviors on Chromium browsers are described below:
- `fullscreen`: All of the available display is used and no UA chrome is shown. This is implemented only for mobile devices running Android or iOS.
- `standalone`: The web app looks like a standalone application. The title bar includes the title of the application, a web app menu button, and caption control buttons (minimize, maximize/restore, close). 
- `minimal-ui`: Similar to `standalone`, except it also contains a back and refresh button. 
- `browser`: Currently, the same as `minimal-ui`

Developers targeting non-mobile devices will find that none of the display modes above offer the ability to create an immersive, native-like title bar for their application. Instead, they must shift content down below the reserved title bar area, which can create a cramped application space especially on portable devices with smaller screens.

This explainer will examine different techniques that could be developed to provide more control of the title bar area to developers while still protecting the rights of users to manage the app window.

## Examples of title bar customization on desktop apps

The title bar area of desktop applications is customized in many popular applications. The title bar area refers to the space to the left or right of the caption controls (minimize, maximize, close etc.) and often contains the title of the application. On Windows, this area can be customized by the developer and apps based on Electron often reclaim this title bar space for frequently used UI like a search box, profile icon, new message icon etc.

### Visual Studio Code
Visual Studio Code is a popular code editor built on Electron that ships on multiple desktop platforms.

This screen shot shows how VS Code uses the title bar to maximize available screen real estate. They include the file name of the currently opened file and the top-level menu structure within the title bar space.

![Visual Studio Code title bar](VSCode.png)

### Spotify
Popular streaming music service Spotify is also built on Electron and they use the title bar space to maximize screen real estate to show the currently signed in user account, a search box and forward/back buttons designed specifically for the Spotify experience.

![Spotify title bar](Spotify.png)

### Microsoft Teams
Workplace collaboration and communication tool Microsoft Teams, also based on Electron for portability, customize the title bar in a similar fashion to Spotify, providing user information, a search and command bar and their own back/forward in-app navigation controls. 

![Microsoft Teams title bar on Mac](MSTeamsMac.png)

## Problem to solve: PWA Title bar area is system reserved

Contrast the above examples of popular desktop applications with the current limitation in the `standalone` display mode in Chromium based desktop PWAs.

![PWA Title bar not available for content](TwitterStandalone.png)

- The UA supplied title bar is styled by the browser (with input from the developer via the manifest's [`"display"`](https://developer.mozilla.org/en-US/docs/Web/Manifest/display) and [`"theme_color"`](https://developer.mozilla.org/en-US/docs/Web/Manifest/theme_color))
- The 3-dot menu is displayed beside the caption controls

None of this area is available to application developers. This is a problem where 
- screen real estate is at a premium when windowed apps have reduced viewport
- the developer is forced to make another area underneath the title bar for the application controls they'd like most prominently displayed
- UA supplied controls cannot be styled or hidden which takes away a developer's ability to fully control the app experience 

## Goals

- Provide a declarative way for developers to have the UA host their installed web app with the title bar area available for their content  
- Ensure accessible user control of the app window is maintained (at minimum - UA supplied minimize, close and drag caption controls)
- The UA respects the caption controls design of the host operating system while adapting to the applications color/theme 


## Proposal

The solution proposed in this explainer is in multiple parts
1. A new member for the web app manifest - `caption_controls_only`
2. New APIs for developers to query the bounding rects and other states of the UA provided caption controls overlay which will overlay into the web content area through a new object on the `Window.menubar` property called `controlsOverlay`
3. A standards-based way for developers to define system drag regions on their content

### Overlaying Caption Controls
To provide the maximum addressable area for web content, the User Agent (UA) will create a frameless window removing all UA provided chrome except for a caption controls overlay.

The caption controls overlay ensures users can minimize, maximize or restore, and close the application. In addition, it will hold the web app menu button which opens a menu similar to the settings menu in a normal Chromium webpage. To the interior of the web app menu button, there will be a small region the same width and height as one of the caption control buttons and will serve as a draggable region to ensure that the window is draggable in the case where the web content doesn't provide any draggable area.

![Caption Controls Overlay on an empty PWA](CaptionControlsOnly.png)

The caption controls overlay will always be on top of the web content's Z order and will accept all user input without flowing it through to the web content.

The coordinate system will not be affected by the overlay, although content my be covered by the overlay.
- The point (0,0) will be the top left corner of the viewport. This point will fall _under_ the overlay if the overlay is in the top-left corner.
- `window.innerHeight` will return the full height of the client area including the area under the overlay. On operating systems which do not include borders around the window, `window.innerHeight === window.outerHeight`
- `vh` and `vw` units would be unaffected. They would still represent 1/100th of the height/width of the viewport which is also not affected by the overlay.

The caption controls overlay would use the `"theme_color"` from the manifest as the background color. When hovered over and clicked, the controls should honor the operating system design behavior.

The desire to place content into the title bar area and use an overlay for the caption controls will be declared within the web app manifest through a new member called `caption_controls_only`. An optional member of boolean type which is false by default and could be used in conjunction with display mode `standalone`. This member will be ignored on Android and iOS, and when used in conjunction with any other `display` modes.

```json
{
  "display": "standalone",
  "caption_controls_only": "true"
}
```

### Working Around the Caption Control Overlay
Web content will need to be aware of the UA reserved area of the caption controls overlay and ensure those areas aren't expecting user interaction. This overlay can be worked around similar to the way developers work around notches in a phone screen.

In the example of Windows operating systems, caption controls are either drawn on the upper right or upper left of the frame depending on which system language is in use:
- Left to right languages - close button shown on the upper right of the frame
- Right to left languages - close button shown on the upper left of the frame

The bounding rectangle of the caption controls overlay will need to be made available to the web content, as well as a property that describes the visibility of the overlay.

To accommodate these requirements, this explainer proposes a new object on the `Window.menubar` property called `controlsOverlay`.

`controlsOverlay` would make available the following objects:
* `getBoundingRect()` which would return a [`DOMRectReadOnly`](https://developer.mozilla.org/en-US/docs/Web/API/DOMRectReadOnly) that represents the area under the caption controls overlay. Interactive web content should not be displayed beneath the overlay.
* `visible` a boolean to determine if the caption controls overlay has been rendered

For privacy, the `controlsOverlay` will not be accessible to iframes inside of a webpage. See [Privacy Considerations](#privacy-considerations) below

### Defining Draggable Regions in Web Content
Web developers will need a standards-based way of defining which areas of their content within the general area of the title bar should be treated as draggable. 

#### Possible Solutions: Standardizing CSS app-region
Chromium based browsers have a prefixed, non-standard CSS property `-webkit-app-region: drag` and `-webkit-app-region: no-drag` that allows developers to markup rectangular regions of their content as draggable. This property is used for full customization of the title bar for Electron based applications [referenced here](https://electronjs.org/docs/api/frameless-window#draggable-region).

Per the Electron documentation, text selection can accidentally occur within draggable regions, so it's recommended to also use the CSS property `user-select: none` on the element to avoid accidental text selection. 

Both of these webkit prefixed properties have been shipping in Chromium for some years and could be leveraged by the UA to provide a solution to this problem. This would require standardizing the app-region property through the CSS working group. 

#### Possible Solutions: Declare a CSS Selector for draggable regions
Within the app manifest file, the developer could declare two selectors that the UA could then use to identify areas that should be treated as a drag and no-drag regions - `DragSelector: dragMe` and `NoDragSelector: dontDragMe`. 

Classes with the `dragMe` selector would then be treated as draggable and have pointer events handled by the host operating environment as drag events on the window itself. Classes with `dontDragMe` will not be draggable, even if they're nested inside of an element with the `dragMe` class.

## Example

Below is an example of how these new features could be used to create a web application with a custom title bar. 

![Example code as a PWA](CustomTitleBarExample.png)

### manifest.webmanifest
In the manifest, set `display: standalone` and `caption_controls_only: true`. Set the `theme_color` to be the desired color of the title bar.
```JSON
{
  "name": "Example PWA",
  "display": "standalone",
  "caption_controls_only": "true",
  "theme_color": "#254B85"
}
```

### index.html
There are two main regions below: the `titleBar` and the `mainContent`. The `titleBar` is set to be `draggable` and the search box inside is set to be `nonDraggable`. 
```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width">
    <title>Example PWA</title>
    <link rel="stylesheet" href="style.css">
    <link rel="manifest" href="./manifest.webmanifest">
  </head>
  <body>
    <div id="titleBar" class="titleBarContainer draggable">
      <span>Example PWA</span>
      <input class="nonDraggable" type="text" placeholder="Search"></input>
    </div>
    <div id="mainContent"><!-- The rest of the webpage --></div>
  </body>
  <script src="app.js"></script>
</html>
```

### style.css
The draggable regions are set using `app-region: drag` and `app-region: no-drag`. 

The title bar is fixed in place with `position: absolute` so that it doesn't scroll out of view. Its background color is the same as the `theme_color` from the manifest to create one seamless title bar. It also sets `user-select: none` to prevent any attempts at dragging the window to be consumed instead by highlighting text inside of the div.

The container for the `mainContent` of the webpage is also fixed in place with `position: absolute`. It sets `overflow-y: scroll` to allow its contents to scroll vertically within the container.
```css
.draggable {
  app-region: drag;
}

.nonDraggable {
  app-region: no-drag;
}

.titleBarContainer {
  position: absolute;
  display: flex;
  user-select: none;
  background-color:#254B85;
  color: #FFFFFF;
  font-weight: bold;
  text-align: center;
}

.titleBarContainer > input {
  flex: 1;
  margin: 8px;
  border-radius: 5px;
  border: none;
  padding: 8px;
}

.titleBarContainer > span {
  margin: auto;
  padding: 0px 16px 0px 16px;
}

#mainContent {
  position: absolute;
  left: 0;
  right: 0;
  bottom: 0;
  overflow-y: scroll;
}
```

### app.js
The new Javascript APIs are used to get the bounds of the caption controls overlay and determine the layout of the `titleBar` element. Since the overlay could live either in the upper-left or upper-right corner of the viewport, the layout calculations must take into consideration both configurations. If the overlay is in the upper-right corner, the `x` coordinate of overlay will be non-zero, so `overlay.x` is used to determine whether the left and right insets should be `0` or `overlay.width`. 

After styling the `titleBar`, the top inset of the `mainContent` needs to be set as well (the rest of the insets are `0` and were already set in `style.css`). Fortunately, this just requires knowing the height of the overlay.

Since these position values are scaled when resizing the browser window--but the caption control overlay will not--each of these values will need to be reset each time the window is resized. 
```javascript
const resizeTitleBar = () => {
  const overlay = window.menubar.controlOverlay.getBoundingRect();

  const titleBar = document.getElementById('titleBar');
  titleBar.style.left = `${overlay.x ? 0 : overlay.width}px`;
  titleBar.style.right = `${overlay.x ? overlay.width : 0}px`;
  titleBar.style.top = '0px';
  titleBar.style.bottom = `${window.innerHeight - overlay.bottom}px`;

  const mainContent = document.getElementById('mainContent');
  mainContent.style.top = `${overlay.height}px`;
}

resizeTitleBar();
window.addEventListener('resize', resizeTitleBar);
```

## Privacy Considerations

Enabling the caption control overlay and draggable regions do not pose considerable privacy concerns other than feature detection. However, due to differing sizes and positions of the caption control buttons across operating systems, the JavaScript API for `window.menubar.controlsOverlay.getBoundingRect()` will return a rect whose position and dimensions will reveal information about the operating system upon which the browser is running. Currently, developers can already discover the OS from the user agent string, but due to fingerprinting concerns there is discussion about [freezing the UA string and unifying OS versions](https://groups.google.com/a/chromium.org/forum/m/#!msg/blink-dev/-2JIRNMWJ7s/yHe4tQNLCgAJ). We would like to work with the community to understand how frequently the size of the caption controls overlay changes across platforms, as we believe that these are fairly stable across OS versions and thus would not be useful for observing minor OS versions.

Although this is a potential fingerprinting issue, it only applies to installed PWAs that use the custom title bar feature and does not apply to general browser usage. Additionally, the `controlsOverlay` API will not be available to iframes embedded inside of a PWA.

## Open Questions

### General
- Would this approach negatively impact coordinate systems? Elements positioned absolutely or fixed? Coordinates returned when querying element or mouse positions via DOM APIs?

### Open Questions: Overlaying Caption Controls
- Dialogs (e.g. permission prompts or `window.alert()`) and overlays (e.g. print or search) that are usually anchored to the top of the client area will be shifted down so that they are vertically anchored to the bottom edge of the caption controls overlay.
  * Where should dialogs (e.g. permission prompts or `window.alert()`) and overlays (e.g. print or search) be anchored? Should they be anchored the top of the window such that they might overlay the caption controls? Or should they be anchored to the bottom of the caption controls overlay so that there is no overlap (this comes with the risk of easy spoofing)?
  * Should the height of the title bar be customizable too?
  * If so, a fixed set of sizes (small, medium, large) or a pixel value that is constrained by the UA?

### Open Questions: Working Around the Caption Control Overlay
- Would it be valuable to an additional member,`window.menubar.controlsOverlay.controls` which has boolean member properties to provide information on which of the caption controls are currently being rendered? This would include `maximize`, `minimize`, `restore`, `close` among other values that are implementation specific, for example a small `dragRegion` area and `settings` menu.  

### Open Questions: Defining Draggable Regions in Web Content
- Different operating systems could have requirements for draggable regions. One approach could be to have a drag region that runs 100% width but only comes down a small number of pixels from the top of the frame. This could provide a consistent area for end users to grab and drag at the cost of reducing the addressable real estate for web content. Is this desirable?
- Could a DOM property on an element be used to identify drag regions on the content?