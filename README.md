# Lock Screen API Explainer

Authors: Louise Brett (loubrett), Glen Robertson (phoglenix)

## Overview

This document proposes a Lock Screen API for enabling web apps on the lock screen of a device. This would allow users to use web apps for tasks such as checking a calendar, using a calculator, taking a photo, or taking a note, all without going through the extra step to unlock their device first. Such an API would be applicable to both desktop and mobile platforms, though it will require some operating system support to implement it.

When apps are run on the lock screen of a device, access to user data should be thoughtfully considered and restricted. For example a camera app running on the lock screen may let the user take photos but not view existing ones, and a calendar app on the lock screen may let the user see an upcoming event but not others. So the proposal includes isolation of lock screen apps, with methods to pass data to and from the lock screen to reduce the risk of apps accidentally exposing user data.

## Proposed API

Apps that want to be shown on the lock screen would indicate this by declaring a URL in their manifest, to be used to launch the app on the lock screen. The URL must be within the _scope_ defined in the manifest. This URL could be a new manifest entry such as _lock\_screen\_start\_url_ but a more generic configuration would allow other custom launch actions to be added in future. We propose adding a _special\_launchers_ list to the manifest, in which items declare a _URL_ and a _purpose_ that may be recognised by user agents.

```
"lock_screen": {
  "start_url": "/some/url" 
}
```

To use the API users would start by calling `window.getLockScreenData()`.

### Passing data between lock screen instance and unlocked instance

Apps running on the lock screen must be isolated from regular user data such as cookies and local storage. This means there will be two instances of the app: the main instance that is used when the device is unlocked, and a separate instance that can be launched on the lock screen without access to user data. The app may use this API to send data from the lock screen instance to the unlocked instance of the app and vice versa.

```js
const data = await window.getLockScreenData();
await data.setData("my-key", "my-content");
```

Ideally, the web app running on the lock screen should not ask the user to authenticate, operating in an anonymous mode and using the lock screen API to sync data to the user's profile. The goal is to avoid accidentally exposing user data to the lock screen by forcing the app to explicitly send any data needed via this API.


### Notifying apps about available data

When the device is unlocked, if there is data available, the instance of the app running in the unlocked context is notified by listening for the `dataitemsavailable` service worker event.

```js
self.addEventListener('dataitemsavailable', event => {
  const processData = async () => {
    const ids = await data.getKeys();
    for (const id of ids) {
      const content = await data.getData(id);
      // ... [Process content]
      await data.deleteData(id);
    }
  };
  event.waitUntil(processData());
});
```

## API Interface

```webidl
alias Key = DOMString;
alias Data = ArrayBuffer;

[SecureContext]
interface LockScreenData {
  // Gets references to all data items available to the app.
  Promise<sequence<Key>> getKeys();

  // Retrieves content of the data item identified by |key|.
  Promise<Data> getData(Key key);

  // Sets or replaces content of a data item.
  Promise<void> setData(Key key, Data data);

  // Deletes a data item.
  Promise<void> deleteData(Key key);
}

// An event named "dataitemsavailable" is dispatched when new data items
// become available.
[Exposed=ServiceWorker]
interface DataItemsAvailableEvent : ExtendableEvent {
  constructor(DOMString type, optional ExtendableEventInit eventInitDict = {});
}
```

## Security and Privacy Considerations

Apps should be careful to expose the minimal user data necessary on the lock screen and it is not recommended to allow users to log in to the app while on the lock screen. User agents could clear data (excluding data in the lock screen API) from the lock screen profile each time it is closed so that if the user had logged in to the app, they would then be logged out. 

It could be worth limiting this API so data can only be transferred one way. Usually this would be to send data from the lock screen instance to the main instance (eg. a camera app) however some apps may prefer to send data in the other direction (eg. a calendar app showing just the title of the next upcoming event). To control this, apps could be required to declare the direction of data transfer in the manifest.

User agents should prevent lock screen apps from navigating outside their scope.

Apps displayed on the lock screen must not be able to masquerade as the system UI for logging in. User agents should display some persistent UI while the lock screen app is open.
