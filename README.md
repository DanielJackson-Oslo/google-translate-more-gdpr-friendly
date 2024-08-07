This is a simple hack to load the Google Translate widget when pressing a button, instead of loading it for all visitors.

If tracking is enabled, it also tracks translations.

It is intended for use in the https://ungfritid.no/ and https://frivillig.no/ universe. 

You probably don't need this, as browser triggered translations are good enough for most use cases, but if you do - here it is! 

# Code walkthrough

The code is in a usable state in the example.html file

There are three elements to the code: 

1. A button that replaces itself with the Google Translate select box, and opens that select box when clicked
2. Scripts to track opening that button, and track all user initiated translations through the browser
3. Styling for that button

## A button that replaces itself

This button:
```html
<div class="translate_box">
  <button 
    onclick="replaceWithTranslateElement()" 
    class="toggle_translate_button" 
    aria-label="Click to load Google Translate and pick a language">
    Translate
  </button>
</div>
```

Calls this script when pressed: 

```javascript
const replaceWithTranslateElement = function() {
    const buttonToReplace = document.querySelector('.toggle_translate_button');
    buttonToReplace.insertAdjacentElement('afterend', loadGoogleTranslateWidget());
    buttonToReplace.remove();
    trackTranslationClick();
};
```

Which replaces the button with the contents of loadGoogleTranslateWidget, a function that creates a series of script tags and injects google translate in the page. 

It also listens for the google translate element to load, then opens the select box from the google translate element automatically. 

This gives the effect that the user is just opening a drop down, even though we just loaded the contents of that drop down from Google.

```javascript
function loadGoogleTranslateWidget() {
    // All script elements must be created with document.createElement, 
    // as browsers do not allow script tags to be inserted with just 
    // innerHTML or similar. Therefore we do this with all the elements,
    // even though it's cumbersome to read and edit. 

    // Create the container element
    const translateWidgetElement = document.createElement('div');
    translateWidgetElement.id = 'googleTranslateWidget';

    // Create the div for the Google Translate Element
    const googleTranslateElement = document.createElement('div');
    googleTranslateElement.id = 'google_translate_element';

    // Create the first script element
    const actiavteGoogleTranslateScripTag = document.createElement('script');
    actiavteGoogleTranslateScripTag.type = 'text/javascript';
    actiavteGoogleTranslateScripTag.async = true;
    actiavteGoogleTranslateScripTag.defer = 'defer';
    actiavteGoogleTranslateScripTag.textContent = `
        function googleTranslateElementInit(){
        var translateElement = new google.translate.TranslateElement(
            {
            pageLanguage:"no",
            autoDisplay:false
            },
            "google_translate_element"
        )
        };
    `;

    // Create the second script element
    const loadGoogleTranslateScripTag = document.createElement('script');
    loadGoogleTranslateScripTag.type = 'text/javascript';
    loadGoogleTranslateScripTag.src = '//translate.google.com/translate_a/element.js?cb=googleTranslateElementInit';
    loadGoogleTranslateScripTag.async = true;
    loadGoogleTranslateScripTag.defer = 'defer';

    // Create the third script element
    const openPickerOnReplacementListenerScript = document.createElement('script');
    openPickerOnReplacementListenerScript.type = 'text/javascript';
    openPickerOnReplacementListenerScript.textContent = /* javascript */`
        var translate_observer = new MutationObserver(function(mutations, observer) {
        mutations.forEach(function(mutation) {
            mutation.addedNodes.forEach(function(node) {
            var selectBox = node.querySelector('.goog-te-combo')
            if (document.contains(selectBox) && typeof selectBox.showPicker == 'function') {
                observer.disconnect();

                var showPickerAfterRender = function() {
                setTimeout(() => {
                    if (selectBox.checkVisibility()) {
                    selectBox.showPicker()
                    } else {
                    showPickerAfterRender()
                    }
                }, 30);
                }
                showPickerAfterRender();
            }
            });
        });
        });

        translate_observer.observe(document.getElementById("google_translate_element"), { childList: true });
    `;

    // Append all elements to the container
    translateWidgetElement.appendChild(actiavteGoogleTranslateScripTag);
    translateWidgetElement.appendChild(loadGoogleTranslateScripTag);
    translateWidgetElement.appendChild(googleTranslateElement);
    translateWidgetElement.appendChild(openPickerOnReplacementListenerScript);

    // Return the translateWidgetElement so it's ready to be inserted:
    return translateWidgetElement;
}
```

## Tracking

We also track button presses and translations:

```javascript
// To track with a different tool, just change the trackEvent function
const trackEvent = function(event, data) { 
    console.log('Tracking ', event, data);
    window.dataLayer = window.dataLayer || [];
    if (window.dataLayer) {
        window.dataLayer.push({event, ...data}); // Google Analytics with GTM
    }
    window.plausible = window.plausible || function() { (window.plausible.q = window.plausible.q || []).push(arguments) };
    if (window.plausible) {
        window.plausible(event, { props: data }); // Plausible analytics
    }
}     

const trackTranslationClick = function() {
    trackEvent("on_site_translate_button_activated")
};

const trackTranslationEvent = function({old_language_attribute, new_language_attribute}) {
    trackEvent("translation_activated", { old_language_attribute, new_language_attribute })
};
```

and set up automatic tracking of translations that are called outside our script, for example from the browser tools: 

```javascript
const setUpTrackingOfTranslations = function() {
    /* Some browsers will mutate the HTML element on translating */
    const htmlElement = document.getElementsByTagName('html')[0];
    /* 
        But others will just mutate all text elements, and we just need to track one 
        we are sure we have. For example the title tag, chosen here */
    const titleElement = document.getElementsByTagName('title')[0];

    /* 
       Most translations will change the lang attribute of the HTML tag,
       but not all. We keep track here. */
    let currentLanguage = htmlElement.getAttribute('lang').toString();

    // Start by checking if the MutationObserver API is available
    if( typeof MutationObserver === 'function') {
        // Tell the observer to monitor for changes to HTML attributes
        var config = { attributes: true };
        // Build the function to run when a change is observed
        var callback = function(mutationList, observer) {
            // Loop through each observed change (using old school loop as ES6 is still not supported in GTM)
            for(var i = 0; i < mutationList.length; i++) {
                // Only do something if the change was on an attribute
                if (mutationList[i]['type'] === 'attributes') {
                    if(
                        // Check for Edge's attributes
                        mutationList[i]['attributeName'] === '_msthash' || mutationList[i]['attributeName'] == '_msttexthash' ||
                        // Check for Google Translate's class
                        (mutationList[i]['attributeName'] === 'class' && mutationList[i]['target'].className.substring(0,9) == 'translate') ||
                        // Check for Safari's lang attribute
                        mutationList[i]['attributeName'] === 'lang'
                    ) {
                        // // If we want to, we can stop observing to only track once per page:
                        // observer.disconnect();
                        
                        const newLanguage = htmlElement.getAttribute('lang').toString();
                        trackTranslationEvent({
                            old_language_attribute: currentLanguage, 
                            new_language_attribute: newLanguage === currentLanguage ? "unchanged_langauge" : newLanguage
                        })

                        currentLanguage = newLanguage;
                    }
                }
                break;
            }
        };
        // Create the actual observer
        var observer = new MutationObserver(callback);
        // Attach the observer to the <title> tag
        observer.observe(titleElement, config);
        // Attach the observer to the <html> tag
        observer.observe(htmlElement, config);
    }
};

setUpTrackingOfTranslations();
```


## Styling

Styling is added to add a little icon to the button and a caret to show it's a drop down now:

```css

.translate_box {
    font-size: 18px;
    --beige: rgb(255, 241, 230);
    --dark-beige: rgb(255, 227, 204);
    --orange: rgb(255, 115, 0);
    --font-family: "Campton W00", sans-serif;

    position: relative;
    width: 8em; /* 144px */
    height: 2em; /* 38px */
    overflow: hidden; 
}

.translate_box::after {
    /* Caret */
    display: block;
    content: url('data:image/svg+xml; utf8, <svg width="14px" height="14px" version="1.1" viewBox="0 0 100 100" xmlns="http://www.w3.org/2000/svg"><path fill="rgb(16, 7, 0)" d="m49.5 78.977c-1.5898 0.003907-3.1172-0.62891-4.2422-1.7578l-37.477-37.477c-2.3398-2.3438-2.3359-6.1367 0.003906-8.4805 2.3398-2.3398 6.1367-2.3438 8.4805-0.003907l33.234 33.234 33.234-33.234c2.3438-2.3398 6.1406-2.3359 8.4805 0.003907 2.3398 2.3438 2.3438 6.1367 0.003906 8.4805l-37.477 37.477c-1.125 1.1289-2.6523 1.7617-4.2422 1.7578z"/></svg>');
    position: absolute;
    height: 1em;
    width: 1em;
    top: 0.4em;
    right: 0.4em;
    pointer-events: none;	
}


.toggle_translate_button,
#googleTranslateWidget select {
    appearance: none;
    -webkit-appearance: none;  /* safari */
    box-sizing: border-box;
    display: block;
    width: 100%;
    text-align: left;
    border-radius: 0px;
    font-family: var(--font-family);
    font-size: 14px;
    line-height: normal;
    background-color: var(--beige);
    padding: 5px;
    border-color: var(--dark-beige);
    border-width: 4px;
    border-style: solid;
    background-image: url("data:image/svg+xml;base64,PD94bWwgdmVyc2lvbj0iMS4wIiBlbmNvZGluZz0iVVRGLTgiPz4KPHN2ZyB3aWR0aD0iMTAwcHQiIGhlaWdodD0iMTAwcHQiIHZlcnNpb249IjEuMSIgdmlld0JveD0iMCAwIDEwMCAxMDAiIHhtbG5zPSJodHRwOi8vd3d3LnczLm9yZy8yMDAwL3N2ZyI+CiA8cGF0aCBkPSJtODAuMDQ3IDE4LjI2MmMtNi4zMTI1LTUuOTYwOS0xNC4xOTEtOS44OTA2LTIyLjgyLTExLjMyOC00LjYwMTYtMC44NTkzOC05LjQ4ODMtMC44OTA2Mi0xNC4xMDUtMC4wODIwMzItMTAuNjkxIDEuNjQ4NC0yMC40MzQgNy4zOTA2LTI3LjI1NCAxNS43ODUtNi4yMDMxIDcuNzIyNy05LjYxNzIgMTcuNDM4LTkuNjE3MiAyNy4zNjNzMy40MTQxIDE5LjY0MSA5LjYxMzMgMjcuMzYzYzguMjgxMiAxMC4yMyAyMC44NCAxNi40NDEgMzQuMTM3IDE2LjM4NyAxMy4zMzYgMCAyNS43NzctNS45NzI3IDM0LjEzMy0xNi4zODcgNi4yMDMxLTcuNzIyNyA5LjYxNzItMTcuNDM4IDkuNjE3Mi0yNy4zNjNzLTMuNDE0MS0xOS42NDEtOS42MTMzLTI3LjM2M2MtMS4yNjE3LTEuNTc0Mi0yLjY0ODQtMy4wMDc4LTQuMDg5OC00LjM3NXptLTUuMDMxMiAyOC42MTNjLTAuMjUzOTEtNS42ODM2LTEuMjEwOS0xMS4yMjMtMi45NTMxLTE2LjQ4NC0wLjMwMDc4LTAuOTEwMTYtMC42NzE4OC0xLjc4NTItMS0yLjY3NTggMi4yMjY2LTAuODc1IDQuMzkwNi0xLjkxNDEgNi40ODQ0LTMuMDc4MSAwLjU4MjAzIDAuNjI4OTEgMS4xNzk3IDEuMjM4MyAxLjcxODggMS45MTQxIDQuNzMwNSA1Ljg5NDUgNy41IDEyLjg2NyA4LjEwNTUgMjAuMzI0em0tNi44MjQyIDMxLjEwMmMxLjQ5NjEgMC41NzQyMiAyLjk2ODggMS4xOTUzIDQuNDEwMiAxLjkxMDItMi42MDU1IDEuOTc2Ni01LjQ1MzEgMy41NzAzLTguNDY4OCA0Ljc5NjkgMS40OTIyLTIuMTUyMyAyLjg1OTQtNC4zODI4IDQuMDU4Ni02LjcwN3ptLTIuMjIyNy0xMC40ODhjLTAuMDExNzE5IDAuMDM1MTU3LTAuMDIzNDM4IDAuMDcwMzEzLTAuMDM1MTU2IDAuMTA1NDcgMC4wMDM5MDYtMC4wMTE3MTkgMC4wMTE3MTgtMC4wMjczNDQgMC4wMTU2MjUtMC4wMzkwNjJoMC4wMDM5MDZjLTAuMzQzNzUgMC45Mzc1LTAuNzAzMTIgMS44MjAzLTEuMTA1NSAyLjcyNjYtOS44MTY0LTIuNjI4OS0yMC4xMTctMi42MTcyLTMwLjA2MiAwLjA5NzY1Ni0wLjM5MDYyLTAuOTEwMTYtMC44MDA3OC0xLjgyNDItMS4xMDU1LTIuNzY5NS0xLjU1NDctNC42Nzk3LTIuNDM3NS05LjUzOTEtMi43MTA5LTE0LjQ4NGgzNy43NzdjLTAuMjgxMjUgNC44ODY3LTEuMTg3NSA5LjcwNy0yLjc3NzMgMTQuMzYzem0tMzAuNzczIDE2LjkxOGMtMi43NjE3LTEuMTg3NS01LjM3NS0yLjY5MTQtNy43ODUyLTQuNTE5NSAxLjMxMjUtMC42NTYyNSAyLjY2MDItMS4yMzQ0IDQuMDMxMi0xLjc3MzQgMS4xMjExIDIuMTc1OCAyLjM4MjggNC4yNjU2IDMuNzUzOSA2LjI5M3ptLTQuMjIyNy0zNy41MzFjMC4yODEyNS00LjkzNzUgMS4xODM2LTkuODI4MSAyLjc3MzQtMTQuNTcgMC4yNDYwOS0wLjkxNDA2IDAuNzA3MDMtMS43Njk1IDEuMDU4Ni0yLjY2OCA0Ljk4ODMgMS4zNTk0IDEwLjA5IDIuMDgyIDE1LjE5NSAyLjA4MiA1LjExMzMgMCAxMC4xODgtMC43MTA5NCAxNS4xMTctMi4wNTQ3IDAuMzQzNzUgMC45MjU3OCAwLjcxMDk0IDEuODYzMyAxLjA0MyAyLjc5NjkgMS40OTYxIDQuNTQ2OSAyLjM0MzggOS4zMzk4IDIuNTk3NyAxNC40MTR6bTAuNDY4NzUtMjQuOTg4Yy0xLjM2MzMtMC41MzUxNi0yLjcxMDktMS4xMDk0LTQuMDQzLTEuNzczNCAyLjM5ODQtMS44MTY0IDUuMDAzOS0zLjMxNjQgNy43NjE3LTQuNS0xLjM1NTUgMi4wMTU2LTIuNjA5NCA0LjA5NzctMy43MTg4IDYuMjczNHptMzcuMDY2IDAuMDIzNDM3Yy0wLjU4NTk0LTEuMTMyOC0xLjE3OTctMi4yNDIyLTEuODMyLTMuMzEyNS0wLjU4MjAzLTEuMDE1Ni0xLjIwNy0yLjAxNTYtMS44NTk0LTMgMi43NjE3IDEuMTkxNCA1LjM3ODkgMi42OTUzIDcuNzkzIDQuNTIzNC0xLjMzMiAwLjY2NDA2LTIuNzA3IDEuMjQ2MS00LjEwMTYgMS43ODkxem0tMTguNDA2LTkuNDEwMmMxLjU2MjUgMC4wMDM5MDYgMy4xMDU1IDAuMTI1IDQuNjMyOCAwLjMxNjQxIDIuNTA3OCAyLjgwMDggNC43MzA1IDUuNzc3MyA2LjU2NjQgOC45NjQ4IDAuNDI5NjkgMC43MDcwMyAwLjc4OTA2IDEuNDEwMiAxLjE2OCAyLjExMzMtOC4xNTYyIDIuMDM5MS0xNi43ODEgMi4wNDMtMjQuOTk2LTAuMDIzNDM3IDIuMTI1LTMuOTgwNSA0LjY5OTItNy42NzU4IDcuNzUzOS0xMS4wMzEgMS42MDU1LTAuMjAzMTIgMy4yMjY2LTAuMzM1OTQgNC44NzUtMC4zMzk4NHptLTI5LjM2MyAxNC4wNTFjMC41NDI5Ny0wLjY3NTc4IDEuMTQ0NS0xLjI4OTEgMS43MjY2LTEuOTIxOSAyLjA4MiAxLjE0ODQgNC4yMTA5IDIuMTcxOSA2LjM4NjcgMy4wMzkxLTAuMzY3MTkgMC45MjU3OC0wLjgxMjUgMS43MjI3LTEuMDYyNSAyLjczNDQtMS43ODkxIDUuMzU5NC0yLjc4NTIgMTAuODkxLTMuMDcwMyAxNi40NzNoLTEyLjA5YzAuNjA1NDctNy40NTcgMy4zNzUtMTQuNDMgOC4xMDk0LTIwLjMyNHptLTguMTA5NCAyNi41NzRoMTIuMDljMC4yODEyNSA1LjY1NjIgMS4yNzczIDExLjIxNSAzLjA2NjQgMTYuNTU1IDAuMzMyMDMgMS4wNTQ3IDAuNjkxNDEgMS43ODkxIDEuMDg1OSAyLjczODMtMi4xOTUzIDAuODgyODEtNC4zMTY0IDEuODgyOC02LjM1OTQgMy4wMTE3LTAuNjAxNTYtMC42NTIzNC0xLjIxODgtMS4yODUyLTEuNzc3My0xLjk4MDUtNC43MzA1LTUuODk0NS03LjUtMTIuODY3LTguMTA1NS0yMC4zMjR6bTM3LjI5MyAzNC4zNzUtMC4zODI4MS0wLjAwMzkwNmMtMS40MjU4LTAuMDE1NjI1LTIuODM5OC0wLjExNzE5LTQuMjM4My0wLjI5Mjk3LTMuMDcwMy0zLjM2MzMtNS42NjAyLTcuMDYyNS03LjgyMDMtMTEuMDc4IDguMTQ4NC0yLjA0NjkgMTYuNjI1LTIuMDYyNSAyNC42NTYtMC4wOTM3NS0yLjE5MTQgNC4wNDY5LTQuODU1NSA3LjgxNjQtOC4wMTU2IDExLjIzNC0xLjM4MjggMC4xNTIzNC0yLjc4NTIgMC4yMzQzOC00LjE5OTIgMC4yMzQzOHptMjkuMzQtMTQuMDUxYy0wLjUzOTA2IDAuNjcxODgtMS4xMzY3IDEuMjgxMi0xLjcxNDggMS45MTAyLTIuMTg3NS0xLjE5OTItNC40Mzc1LTIuMjY1Ni02LjczNDQtMy4xNjQxIDAuMzgyODEtMC44NjMyOCAwLjczMDQ3LTEuNzM4MyAxLjA1NDctMi42MjExIDEuODAwOC01LjI3MzQgMi44MjAzLTEwLjc0NiAzLjEyMTEtMTYuNDQ5aDEyLjM4M2MtMC42MDU0NyA3LjQ1Ny0zLjM3NSAxNC40My04LjEwOTQgMjAuMzI0eiIvPgo8L3N2Zz4K");
    background-position: 5px center;
    background-repeat: no-repeat;
    background-size: 1.42em 1.42em;
    padding-left: 2.2em;
    padding-right: 1.8em;
}

.toggle_translate_button:hover,
#googleTranslateWidget select:hover {
    border-color: var(--orange);
}


#googleTranslateWidget select::-ms-expand {
    display: none; /* Removes arrow in IE */
}

#googleTranslateWidget .goog-te-gadget {
    font-family: var(--font-family);
}

#googleTranslateWidget .goog-te-gadget a {
    display: none; /* Hides the google logo, so it can't be accessed by tab-ing. That Google is the translator is still prominently displayed  */
}

#googleTranslateWidget select.goog-te-combo {
    margin: 0 0 10px 0;
}

```
