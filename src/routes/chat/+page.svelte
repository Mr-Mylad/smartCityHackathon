<script>
    // General stuff
    import { goto } from "$app/navigation";
    import { onMount } from "svelte";
    import UserMessage from "$lib/components/UserMessage.svelte";
    import AiMessage from "$lib/components/AiMessage.svelte";

    // Gemini
    import { GoogleGenerativeAI } from "@google/generative-ai";

    // Custom functions
    import { initialAnalysis } from "$lib";

    const genAI = new GoogleGenerativeAI(import.meta.env.VITE_GEMINI_API_KEY);
    const client = genAI.getGenerativeModel({ model: "gemini-1.5-flash" });

    let location;
    let inputValue = "";
    let inputElement;
    let initialAnalysisResult = null;
    let initialAnalysisLoading = true;
    onMount(() => {
        inputElement = document.getElementById("userInput");
        inputElement.focus(); 
        location = localStorage.getItem("location");
        if (!location) goto ("/");
        messageLog.push({role: "user", parts: [{text: `Hey! I'd like to know about the smart city practices in ${location}!`}]});
        messageLog = messageLog;
        localStorage.removeItem("location");
        window.addEventListener("keydown", () => inputElement.focus());
    });


    // CHAT TIME

    let messageLog = [];
    let chat; 

    // UNCOMMENT WHEN HAVE TO RUN

    $: if (location) {
        GetInitialAnalysis().then(result => {
            initialAnalysisResult = result;
            initialAnalysisLoading = false;
        })
    }
    async function GetInitialAnalysis() {
        const result = await initialAnalysis(client, location);
        messageLog.push({role: "model", parts: [{text: result}]});
        messageLog = messageLog;
        chat = client.startChat({
        history: [{
            role: "user", parts: [{text: `You are an AI assistant focused on smart city solutions. When a user selects a city, provide:

    1. A brief overview of existing smart city practices related to sustainability, transportation, and technology.
    2. Suggestions for future implementations, especially in sustainability and technology.
    3. Keep the response clear and actionable, inviting the user to ask follow-up questions for more details or specific examples.

    If city-specific data is unavailable, offer general smart city examples from similar urban areas.

    Start:

    Hey! I'd like to know about ${location}`}]
        }, messageLog[1]]
    });
        return result;
    }




    let sentMessageResponded = true;
    let lastReqErr = false;
    function sendMessage() {
        if (!sentMessageResponded) return;
        if (lastReqErr) {
            messageLog.pop();
            messageLog.pop();
            messageLog = messageLog;
        }
        messageLog.push({
            role: "user",
            parts: [{text: inputValue}]
        });
        messageLog.push({
            role: "model",
            parts: [{text: "..."}]
        })
        messageLog = messageLog;
        sentMessageResponded = false;
        const question = inputValue;
        inputValue = "";
        inputElement.disabled = true;
        document.querySelector("#submitButton").disabled = true;
        chat.sendMessage(question).then(response => {
            messageLog.pop();
            messageLog.push(response.response.candidates[0].content);
            messageLog = messageLog;
            inputElement.disabled = false;
            document.querySelector("#submitButton").disabled = false;
            sentMessageResponded = true;
        }).catch(err => {
            messageLog.pop();
            messageLog.push(`Sorry, there was an error. (P.S., this error will reset on your next message) (details: \n\n${err})`)
            messageLog = messageLog;
            inputElement.disabled = false;
            document.querySelector("#submitButton").disabled = false;
            lastReqErr = true;
            sentMessageResponded = true;
        })
    }
</script>

<div id="chatlog" class="bg-slate-800 rounded-md fixed left-0 top-0 p-4 pb-24 w-full h-full overflow-y-scroll flex flex-col gap-4">
    <UserMessage message={`Hey! I'd like to know about the smart city practices in ${location}!`}></UserMessage>
    {#if initialAnalysisLoading}   
        <AiMessage message="..."></AiMessage>
    {:else if !initialAnalysisLoading}
        <AiMessage message={initialAnalysisResult}></AiMessage>
    {/if}

    {#each messageLog as message, i}
        {#if i > 1}
            {#if message.role === "user"}
                <UserMessage message={message.parts[0].text}></UserMessage>
            {:else}
                <AiMessage message={message.parts[0].text}></AiMessage>
            {/if}
        {/if}
    {/each}
</div>

<form on:submit={sendMessage}>
    <input type="text" class="bg-white rounded-lg p-2 border-4 border-black bottom-4 left-2 absolute w-96" required id="userInput" bind:value={inputValue}>
    <button class="rounded-lg border-4 border-black left-[25rem] absolute w-32 p-2 bg-white bottom-4" id="submitButton">Submit</button>
</form>
<h1 class="text-red-700 text-sm font-bold absolute bottom-0 right-0">PLEASE DO NOT REFRESH THE PAGE, OTHERWISE YOU WILL LOSE ALL OF THE CHAT HISTORY</h1>