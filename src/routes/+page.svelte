<script lang='ts'>
    import { ChatOpenAI } from 'langchain/chat_models/openai'
    import { HumanMessage, SystemMessage } from 'langchain/schema'
    import { PUBLIC_OPENAI_API_KEY, PUBLIC_ELEVEN_LABS_API_KEY } from '$env/static/public'
	import { onMount } from 'svelte';
    import { Avatar, popup } from '@skeletonlabs/skeleton';
	import type { PopupSettings } from '@skeletonlabs/skeleton'
    import { slide } from "svelte-legos"

    const keyword = "toby"

    let recognition: any
    let listening = false
    let transcription: any 
    let message = false
    let responseText = ''
	let permission = true

    // initiate recording
    const startRecordingProcess = () => {
        if (!listening) startListening()
        else stopListening()
    }

    const startListening = () => {
        console.log("Start listening...")
        recognition.start()
        listening = true
    }

    const stopListening = () => {
        console.log("Stop listening...")
        recognition.stop()
        listening = false
    }

    // handle silence and detection

    // communicate with OpenAI
    const getOpenAIResponse = async (msg: string) => {
        console.log("Communicating with OpenAI...")
        const apiKey = PUBLIC_OPENAI_API_KEY
        const chat = new ChatOpenAI({
            openAIApiKey: apiKey
        })
        const response = await chat.call([
            new SystemMessage("You are a helpful voice assistant"),
            new HumanMessage(msg)
        ])
        return response.text
    }

    // convert response to audio using Eleven Labs
    const convertResponseToAudio = async (text: string) => {
        console.log("Converting response to audio...")

        const apiKey = PUBLIC_ELEVEN_LABS_API_KEY
        const options = {
            method: 'POST',
            headers: {
                'Accept': 'audio/mpeg',
                'Content-Type': 'application/json',
                'xi-api-key': apiKey,
            },
            body: JSON.stringify({
                "text": text,
                "model_id": "eleven_monolingual_v1",
                "voice_settings": {
                "stability": 0.5,
                "similarity_boost": 0.5
                }
            })
        }

        // Toby voice
        const voice_id = 'J3i20pjnyJJRyaCPN8nV'

        const response = await fetch(
            `https://api.elevenlabs.io/v1/text-to-speech/${voice_id}`,
            options
        );

        const blob = await response.blob()
        const url = URL.createObjectURL(blob);
        
        const audio = new Audio();
        audio.src = url;
        audio.play();
    }

    onMount(() => {
        const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition;
        recognition = new SpeechRecognition();

		if (typeof chrome !== 'undefined' && chrome.permissions) {
			// Call permissions api
			// chrome.permissions.request({
			// 	permissions: ['audioCapture'] 
			// }, granted => {
			// 	if (granted) {
			// 		// start speech recognition
			// 		console.log("Access granted")
			// 	}
			// });

			chrome.permissions.getAll(
				(result) => {
					console.log(result)
				}
			)

			// check if microphone & sound access enabled
			// chrome.permissions.contains({
			// 	permissions: ['audioCapture'],
			// 	},
			// 	(result: boolean) => {
			// 		if (result) {
			// 			// mic access allowed
			// 			console.log("Access allowed")
			// 			return		
			// 		} 
			// 		else {
			// 			// mic access blocked
			// 			console.log("Access blocked")
			// 			permission = false
			// 		}
			// 	}
			// );
		} 
		else {
			// Handle regular web page
			console.log('Not in extension context');
		}

        recognition.onresult = async (event: any) => {
            transcription = event.results[0][0].transcript;
            console.log("Transcription: " + transcription)

            message = true
            setTimeout(() => {
                message = false
            }, 3000)

            if (transcription && transcription.toLowerCase().includes(keyword)) {
                console.log("Keyword detected...")
                responseText = await getOpenAIResponse(transcription)
                await convertResponseToAudio(responseText)
                console.log("Playing audio...")

                setTimeout(() => {
                    responseText = ''
                }, 3000)
            }

            stopListening()
        };
    })

	// manually enable microphone & sound access
	const handlePermission = () => {
		chrome.permissions.contains({
			permissions: ['audioCapture'],
			},
			(result: boolean) => {
				if (result) {
					// mic access allowed
					return		
				} 
				else {
					// mic access blocked
					chrome.tabs.create({
						url: 'chrome://settings/content/siteDetails?site=chrome-extension://ngjfhcicijldancmdehgiegagbnhpddp'
					});
				}
			}
		);
	}

    const popupHover: PopupSettings = {
        event: 'hover',
        target: 'popupHover',
        placement: 'left'
    };

</script>

<div class="h-full w-full p-4 relative">

    <!-- Since chrome API not working, show this notification always for now -->
    <div class="w-[600px] m-8 mx-auto mt-4 text-center">
        <p>Please enable 'Microphone (Allow)' and 'Sound (Allow)' in settings.</p>
        <button type="button" class="btn variant-filled my-4"
            on:click={handlePermission}>
            <span>⚙️</span>
            <span>Go to Settings</span>
        </button>
    </div>

    {#if !permission}

        <div class="w-[600px] m-8 mx-auto text-center">
            <p>Please enable 'Microphone (Allow)' and 'Sound (Allow)' in settings.</p>
            <button type="button" class="btn variant-filled my-4"
                on:click={handlePermission}>
                <span>⚙️</span>
                <span>Go to Settings</span>
            </button>
        </div>

    {:else}

        <div class="absolute bottom-4 right-4 cursor-pointer z-[999] {listening && "recording"}" 
            on:click={startRecordingProcess}
            on:keyup={() => {}}
            use:popup={popupHover}>
            <Avatar class="shadow" src="https://oliverai.s3.amazonaws.com/assets/avatars/idea.png" width="w-12" 
                background="{listening ? "bg-rose-500" : "bg-surface-400-500-token"}"/>
        </div>

        <div class="card p-4 variant-filled-error z-[999]" data-popup="popupHover">
            {#if !listening}
                <p>Say "Hey Toby" and ask question!</p>
            {:else}
                <p>Toby is listening...</p>
            {/if}
            <div class="arrow variant-filled-error" />
        </div>

        {#if message}
            <div class="card p-4 variant-filled-tertiary z-[999] absolute bottom-32 right-3"
                transition:slide={{ delay: 500, duration: 1000, direction: 'bottom' }}>
                <p>{transcription}</p>
            </div>
        {/if}

        {#if responseText !== ''}
            <div class="card p-4 variant-filled-error z-[999] absolute bottom-52 right-3"
                transition:slide={{ delay: 1000, duration: 1000, direction: 'bottom' }}>
                <p>{responseText}</p>
            </div>
        {/if}

    {/if}

</div>

<style>
    @keyframes pulse {
        0% { 
            transform: scale(1);
        }
        50% {
            transform: scale(1.3);
        }
        100% {
            transform: scale(1);
        }
    }

    .recording {
        animation: pulse 1s infinite;
    }

</style>