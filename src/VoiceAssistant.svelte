<script>
  import axios from "axios";
  import process from 'process';
  import { onMount } from 'svelte';

  //const API_KEY = process.env.OPENAI_API_KEY;
  //console.log("open_aikey" + process.env.OPENAI_API_KEY + "apik" + API_KEY)

  let recognizedText = "";
  let assistantResponse = "";
  let conversationHistory = [];
  let voices = [];
  let selectedVoice = null;
  let isListening = false;
  let recognitionInstance = null;
  let recognizedTextParts = [];

  const sendInitialMessage = async () => {
    const initialResponse = await getResponseFromOpenAI("hello");
    console.log("Initial Response:", initialResponse);
     
  };

  
  onMount(() => {
    sendInitialMessage();
  });

  const startRecognition = () => {
    const SpeechRecognition =
      window.SpeechRecognition || window.webkitSpeechRecognition;
    const recognition = new SpeechRecognition();
    recognition.lang = "en-UK";
    recognition.interimResults = true;
    recognition.continuous = true;

    recognition.onresult = async (event) => {
      const lastResultIndex = event.results.length - 1;
      const text = event.results[lastResultIndex][0].transcript;

      if (event.results[lastResultIndex].isFinal) {
        recognizedText = text;
      }
    };
    recognition.onerror = (event) => {
      console.error("Speech recognition error:", event.error);
    };

    recognition.start();
    recognitionInstance = recognition;
    isListening = true;
  };

  const stopRecognition = async () => {
    if (recognitionInstance) {
      recognitionInstance.stop();
      isListening = false;

      if (recognizedText.trim() !== "") {
        const response = await getResponseFromOpenAI(recognizedText);
        assistantResponse = response;
        speak(response);

        conversationHistory = [
          ...conversationHistory,
          { role: "user", text: recognizedText },
          { role: "assistant", text: response },
        ];
      }

      recognizedText = ""; // clear recognized text after processing
    }
  };

  const sleep = (ms) => new Promise((resolve) => setTimeout(resolve, ms));

  const getResponseFromOpenAI = async (text, retries = 3, retryDelay = 1000) => {
    try {
      const recentConversationHistory = conversationHistory.slice(-6); // Get last 6 turns (3 questions and 3 answers)

      const messages = [
        {
          role: "system",
          content:
            "You are an  English teacher. Answer the questions and provide guidance with proper grammar, vocabulary, and usage in simpler terms.",
        },
        ...recentConversationHistory.map((turn) => ({
          role: turn.role === "user" ? "user" : "assistant",
          content: turn.text,
        })),
      ];
      // Add   latest message and add it to the messages array
      messages.push({ role: "user", content: `Briefly elaborate:- ${text}` });

       
      const response = await axios.post(
        "https://api.openai.com/v1/chat/completions",
        {
          model: "gpt-3.5-turbo",
          messages,
          temperature: 0.7,
          max_tokens: 100,  
        },
        {
          headers: {
            "Content-Type": "application/json",
            Authorization: `Bearer `,
          },
        }
      );

       
      console.log("API Response:", response.data);  
    const summarizedResponse = response.data.choices[0].message.content.trim();
    console.log("Summarized Response:", summarizedResponse);
      return summarizedResponse;
    } catch (error) {
      if (error.response && error.response.status === 429) {
        if (retries > 0) {
          console.warn(
            `Rate limit reached. Retrying in ${retryDelay}ms (remaining retries: ${retries})...`
          );
          await sleep(retryDelay);
          return await getResponseFromOpenAI(text, retries - 1, retryDelay * 2);
        } else {
          console.error(
            "Error getting response from OpenAI: rate limit reached, no more retries."
          );
          return "";
        }
      } else {
        console.error(
          "Error getting response from OpenAI:",
          error,
          JSON.stringify(error, null, 2)
        );

        return "";
      }
    }
  };

  const speak = (text) => {
    const synth = window.speechSynthesis;
    const availableVoices = synth.getVoices();
    const prioritizedVoices = {
      Chrome: ["Google US English", "Google UK English Female"],
      Edge: [
        "Microsoft Sonia Online (Natural) - English (United Kingdom)",
        "Microsoft Clara Online (Natural) - English (Canada)",
      ],
    };

    const browser = window.navigator.userAgent.includes("Chrome")
      ? "Chrome"
      : "Edge";
    const voiceList = prioritizedVoices[browser];

    const selectedVoice = availableVoices.find((voice) =>
      voiceList.includes(voice.name)
    );

    const sentences = text.split(". ").map((s) => s.trim());

    const speakSentence = (index) => {
      if (index >= sentences.length) {
        return;
      }

      const sentence = sentences[index];
      const utterance = new SpeechSynthesisUtterance(sentence);

      if (selectedVoice) {
        utterance.voice = selectedVoice;
      } else {
        console.log("No suitable voice found, using default voice.");
      }
      utterance.lang = "en-GB";
      utterance.rate = 1;  
      utterance.pitch = 1;  
      utterance.volume = 1;  

      utterance.onend = () => {
        setTimeout(() => {
          speakSentence(index + 1);
        }, 100);
      };

      synth.speak(utterance);
    };

    speakSentence(0);
  };

  const stopSpeaking = () => {
    const synth = window.speechSynthesis;
    synth.cancel();
  };

  const repeatLastAssistantText = () => {
    const lastAssistantMessage = conversationHistory
      .slice()
      .reverse()
      .find((turn) => turn.role === "assistant");

    if (lastAssistantMessage) {
      speak(lastAssistantMessage.text);
    } else {
      console.log("No previous assistant message found to repeat.");
    }
  };
</script>


<style>
  main {
    display: flex;
    flex-direction: column;
    align-items: center;
    padding: 20px;
    max-width: 600px;
    margin: auto;
    gap: 20px;
  }

  h1 {
    color: #4a4e69;
  }

  .conversation-history {
    width: 100%;
    border: 1px solid #bcbcbc;
    padding: 20px;
    border-radius: 8px;
    max-height: 300px;
    overflow-y: auto;
  }

  .message {
    background-color: #f4f4f4;
    padding: 10px;
    border-radius: 5px;
    margin: 5px 0;
    border-left: 5px solid #4a4e69;
  }

  .message.assistant {
    background-color: #d0e8f2;
    border-color: #82c4e6;
  }

  button {
    padding: 10px 20px;
    border: none;
    background-color: #4a4e69;
    color: white;
    border-radius: 5px;
    cursor: pointer;
    transition: background-color 0.3s;
  }

  button:hover {
    background-color: #3b3f5c;
  }

  button:disabled {
    background-color: #cccccc;
    cursor: not-allowed;
  }
</style>

<main>
  <h1>Voice Assistant</h1>

  <div class="conversation-history">
    <h2>Conversation History:</h2>
    {#each conversationHistory as entry, index}
      <p class={`message ${entry.role}`}>
        <strong>{entry.role === "user" ? "You" : "Assistant"}:</strong>
        {entry.text}
      </p>
    {/each}
    <p>Recognized Text: {recognizedText}</p>
  </div>

  <button
    class="fixed-size-button"
    on:mousedown={startRecognition}
    on:mouseup={stopRecognition}
    on:touchstart={(e) => {
      e.preventDefault();
      startRecognition();
    }}
    on:touchend={(e) => {
      e.preventDefault();
      stopRecognition();
    }}
  >
    {isListening ? "Listening..." : "HOLD & SPEAK"}
  </button>

  <button class="fixed-size-button" on:click={stopSpeaking}>Stop</button>

  <button on:click={repeatLastAssistantText}>Repeat Text</button>
</main>