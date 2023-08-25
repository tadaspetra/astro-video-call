# Build a Video Call with Astro

Astro is a framework that renders static HTML to your website with no JavaScript unless you explicitly ask for it and it is gaining popularity fast. I wonder if you can add video calls to your Astro website?

The answer is yes! This article will teach you exactly how you can do it.


## What We Will Use
Of course, we need Astro, since it is the title of this article. 

The second part of the title is about video calls. For that, we will use Agora. Specifically, we will use the `agora-rtc-react` package. This package is a React SDK for interfacing with [Agora](https://docs.agora.io/en/video-calling/get-started/get-started-sdk?platform=react-js).

Currently, there is no native Astro package. But Astro was built specifically to be used with almost any popular web framework. So the `agora-rtc-react` will work just fine.

We will also use Tailwind CSS to do some minor styling of the application.

## Create an Agora Account
[Sign up](https://sso2.agora.io/en/v6/signup) for an Agora account and log in to the dashboard.

Navigate to the Project List tab under the Project Management tab, and create a project by clicking the blue Create button.

Retrieve the App ID, which we’ll use to authorize the app requests as we develop the application.

## Initialize the Astro Project
1. Create your Astro project using `npm create astro@latest`. 
2. Add Tailwind CSS using `npx astro add tailwind`.
3. Add React using `npx astro add react`.
4. Add the Agora UI Kit using `npm install agora-rtc-react`.
5. Add `PUBLIC_AGORA_APP_ID = '<---Your App Id--->'` to your `.env` file.

## Structure of the Project 
```
├── src
│   ├── components
│   │    └── Call.tsx
│   ├── layouts
│   │   └── Layout.astro
│   └── pages
│       ├── index.astro
│       └── channel
│           └── [channelName].astro
├── public
│   └── favicon.png
├── package.json
├── astro.config.js
├── tsconfig.json
└── tailwing.config.cjs
```

## Building a Form with Astro

### Astro SSG vs SSR
By default Astro projects use a static site generator (SSG) that creates a static site which you cannot change. Astro provides a way to interface with different frameworks (React, Svelte, Vue, etc.) using a mechanism called islands. Forms need to be interactive, and we could definitely use an island to get this done. However it is possible to create a form with Astro that ships with zero JavaScript.

To do that, we need to enable the server-side renderer (SSR) by adding `output: 'server'` to the `astro.config.mjs` file. This means that our pages will get rendered on the server. This brings the benefits of being able to access server-side data and to run some functions before the page loads. 

### Form UI
For the case of creating and submitting a form, we will load the page initially to show the form. Then whenever we submit the form, it will act much like an API endpoint. 

Normally, when you submit a form the data will be added to the URL, and you would have to parse it out. By changing the form's data transfer method to POST, it will send the form data as part of the `Request` body. For this application the only data we need is the name of the channel the user is trying to join:

```html
<div class="flex flex-col items-center">
	<h1
		class="mb-4 mt-20 text-4xl font-extrabold leading-none tracking-tight text-gray-900"
	>
		<span class="text-orange-500">Astro</span> x <span class="text-blue-500"
			>Agora</span
		>
	</h1>
	<form method="POST">
		<p>{error}</p>
		<div class="md:flex md:items-center mt-6">
			<div>
				<label
					class="block text-gray-500 font-bold md:text-right mb-1 md:mb-0 pr-4"
					for="inline-full-name"
				>
					Channel Name
				</label>
			</div>
			<div>
				<input
					class="bg-gray-200 appearance-none border-2 border-gray-200 rounded w-full py-2 px-4 text-gray-700 leading-tight focus:outline-none focus:bg-white focus:border-blue-500"
					id="inline-full-name"
					type="text"
					name="channelName"
				/>
			</div>
		</div>
		<div class="text-center">
			<button
				class="inline-flex items-center justify-center px-5 py-3 mt-5 text-base font-medium text-center text-white bg-blue-400 rounded-lg hover:bg-blue-500 focus:ring-4 focus:ring-blue-300 dark:focus:ring-blue-900"
				>Submit</button
			>
		</div>
	</form>
</div>
```

Our form should look like this:
![Enter Channel](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/a1d4dmnspwms1g0f3sym.png)


## Retrieve Form Data and Redirect to Call Page
Once the form is submitted with the data in the `Request`, the page will be reloaded and the data will be sent to the server. This data can be retrieved server-side and acted on before the page is displayed. In this case, we will check for errors. If we find any, we will render the same page again to show the user the error. If there is no error, we will redirect the user to the page associated with the video call channel they entered:

```ts
let error: String = "";

if (Astro.request.method === "POST") {
	try {
		const data = await Astro.request.formData();
		const channelName = data.get("channelName");
		return Astro.redirect(`/channel/${channelName}`);
	} catch (error) {
		if (error instanceof Error) {
			error = error.message;
		}
	}
}
```

## Call Page
With in the call component that we will build with the `agora-rtc-react`, we want to print out the name of the channel we have joined.

The redirect we used in the previous section is dynamic, and it will be different depending on the channelName we entered. To handle this in Astro, we need to create a `channel` folder and in the folder define a file named `[channelName].astro`. The square brackets signify that the dynamic `${channelName}` in our URL will be passed as a parameter in this component. We just need to retrieve it and display it in our UI.

### `client:only` directive
For our Call component we pass the `channelName` and `appId`. But we also need to add a `client:only="react"` directive. This directive is needed for creating an Astro island. As mentioned, Astro is generating static HTML code for the site, and the video call that we create needs to be interactive. 

So without this directive Astro will try to render static HTML from this React component. However by using `client:only` we skip the server-side rendering, and just let React do what React wants to do: render only on the client. 

Since Astro supports many frameworks we also need to specify which framework it needs to us:

```astro
---
import Call from "../../components/Call.tsx";

const { channelName } = Astro.params;
---

<div class="static m-0 p-0">
    <p class="absolute z-10 mt-2 ml-12 text-2xl font-bold text-gray-900">
        {channelName!}
    </p>
    <Call
        client:only="react"
        channelName={channelName!}
        appId={import.meta.env.PUBLIC_AGORA_APP_ID}
    />
</div>
```

## Add Call Component
This last component is a pure React component. The call component will contain three key components:
* Videos of all the participants
* Channel name
* End call button

The channel name is simple a text at the top left of the screen and the call button is at the middle bottom of the screen. Whenever you click the button, it will take you back to the previous screen where you can join a different video call.

The interesting part is displaying the videos of all the participants. In order to do that we need to create an `AgoraRTCProvider`, which initializes and gives us access to the Agora RTC service. Inside this we can now display the video.

### Videos
The videos component is where all will be the part of the site that displays the videos of all the participants. There are many hooks used to set up the call:
* useLocalMicrophoneTrack() retrieves the current user's microphone
* useLocalCameraTrack() retrieves the current user's video input
* useRemoteUsers() retrieves all the user information for the remote users
* useRemoteAudioTracks() retrieves the audio for those users.
* usePublish() publishes the current user's video and audio
* useJoin() joins the actual channel for the video call

```tsx
const { AppID, channelName } = props;
const { isLoading: isLoadingMic, localMicrophoneTrack } = useLocalMicrophoneTrack();
const { isLoading: isLoadingCam, localCameraTrack } = useLocalCameraTrack();
const remoteUsers  = useRemoteUsers();
const { audioTracks } = useRemoteAudioTracks(remoteUsers);

usePublish([localMicrophoneTrack, localCameraTrack]);
useJoin({
	appid: AppID,
	channel: channelName,
	token:  null ,
});
```

Then we need to make sure all the audio for the remote users is started.

```tsx
audioTracks.map((track) => track.play());
```

And lastly we need to define our UI. First a loading state, while we wait for the local user's microphone and video to begin, and then a grid of all the users that are in the call.

```tsx
const deviceLoading = isLoadingMic || isLoadingCam;
if (deviceLoading) return <div className="flex flex-col items-center pt-40">Loading devices...</div>;

const numUsers = remoteUsers.length + 1; 
let numCols = 1;
let numRows = 1;
switch (numUsers) {
case 1:
	numCols = 1;
	numRows = 1;
	break;
case 2:
	numCols = 2;
	numRows = 1;
	break;
case 3:
	numCols = 3;
	numRows = 1;
	break;
case 4:
	numCols = 2;
	numRows = 2;
	break;
default:
	break;
}

return (
<div className="flex flex-col justify-between w-full h-screen p-1">
	<div className={`grid grid-cols-${numCols} grid-rows-${numRows} gap-1 flex-1`}>
	<LocalVideoTrack track={localCameraTrack} play={true} className="w-full h-full" />
	{remoteUsers.map((user) => (
		<RemoteUser user={user} />
	))}
	
	</div>
</div>
);
```

Our final video call should look like this:
![Video Call](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/toidmib42ycml33600i6.png)



## Summary
With that, we have a complete video call experience. Here's how we built it:

1. Create our Astro project.
2. Install Tailwind CSS, React, and the Agora SDK.
3. Create a form to input the channel name.
4. Redirect the site to a URL with the channel name.
5. Display the channel name with a video call.

The code for this project can be found [here](https://github.com/tadaspetra/astro-video-call). And you can find out more about Agora video calling [here](https://docs.agora.io/en/video-calling/get-started/get-started-uikit?platform=reactjs).

Thank you for reading!