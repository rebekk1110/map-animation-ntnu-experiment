<script>
  import { onMount, createEventDispatcher } from "svelte";
  export let participantId;
  export let level;
  export let totalQuestions = 3;
  export let changeCondition;
  export let originalColor;
  export let demoMode = false;


  // For the change condition question
  let change_response = "";      // "Change" or "No Change"
  let change_confidence = ""; 
  let color_confidence = "";  // Scale values as a string (e.g., "5")
  let color_response = "";       
  let submitted = false;
  let redSquareShownTime = 0;

  const dispatch = createEventDispatcher();

  // Added: control whether the survey content is visible.
  // In demoMode, survey content will be hidden until "redSquareShown" event fires.
  let showSurvey = demoMode ? false : true;
  
  onMount(() => {

    if (demoMode) {
      document.addEventListener("redSquareShown", () => {
        showSurvey = true;
      });

    }
    else {
      document.addEventListener("redSquareShown", () => {
        redSquareShownTime = performance.now();
      });
    }

  });
  
  function submitAndNext() {
    // Validate first question
    if (!change_response) {
      alert("🚨 Vennligst svar på om regionen endret klasse.");
      return;
    }
    if (!change_confidence) {
      alert("🚨 Vennligst oppgi hvor sikker du er på ditt svar.");
      return;
    }
    // If the answer is "Change", ensure the original color is selected
    if (change_response === "Change" && !color_response) {
      alert("🚨 Vennligst velg den opprinnelige klassen.");
      return;
    }

    let reactionTime = 0;
    if (redSquareShownTime) {
      reactionTime = performance.now() - redSquareShownTime;
    }


    const data = {
      participant_id: participantId,
      question_id: `Q${level}`, // Single question ID for this level.
      change_response: change_response,
      change_condition: changeCondition,
      change_confidence: parseInt(change_confidence),
      reaction_time: demoMode ? 0 : reactionTime,
      // For color fields, only include values if the answer is "Change".
      original_color: change_response === "Change" ? originalColor : null,
      color_response: change_response === "Change" ? color_response : "",
      color_confidence: change_response === "Change" ? parseInt(color_confidence) : 0
      
    };

    if (demoMode) {
      console.log("Demo mode: simulated submission", data);
      setTimeout(() => {
        submitted = true;
        setTimeout(() => {
          submitted = false;
          // Reset fields for next level
          change_response = "";
          change_confidence = "";
          color_response = "";
          color_confidence = "";
          // Dispatch next event to signal the parent to move on
          document.dispatchEvent(new CustomEvent("nextStep"));
        }, 500);
      }, 1000);
    } else {
      fetch("https://change-blindness-web.onrender.com/submit", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(data)
        
      })
        .then(response => response.json())
        .then(result => {
          console.log("Data: ", data);
          console.log("Full response:", result);
          console.log("Response submitted:", result.message);
          submitted = true;
          setTimeout(() => {
            submitted = false;
            document.dispatchEvent(new CustomEvent("resetAnimation"));
            change_response = "";
            change_confidence = "";
            color_response = "";
            color_confidence = "";
            //document.dispatchEvent(new CustomEvent("nextLevel"));
            if (level < totalQuestions) {
              document.dispatchEvent(new CustomEvent("nextLevel"));
            } else {
              // Last question: mark the study as completed.
              fetch("https://change-blindness-web.onrender.com/complete", {
                method: "POST",
                headers: { "Content-Type": "application/json" },
                body: JSON.stringify({ participant_id: participantId })
              })
                .then(() => {
                  console.log("Study marked as complete!");
                  document.dispatchEvent(new CustomEvent("nextLevel"));
                })
                .catch(error => {
                  console.error("Error marking study as complete:", error);
                });
            }
          }, 1000);
        })
        .catch(error => {
          console.error("Error submitting response:", error);
        });
    }
  }
</script>


<div class="content-wrapper">

  <div class="survey-container">
    {#if demoMode === true}
    <p class="progress-text">Øve-oppgave</p>
     {:else}
    <p class="progress-text">Oppgave {level} av {totalQuestions}</p>
     {/if}
    <div class="task-description">
      <p>
        Observer kartet til venstre.
        Når animasjonen starter vil noen regioner <strong>skifte klasse</strong> (farge).  
        <br>
       Etter animasjonen vil en <strong>rød ramme</strong> fremheve én region. Du skal da svare på om akkuratt denne regionen har skiftet klasse.
      </p>
    </div>
    {#if showSurvey}
    <!-- Question Box 1: Change Response & Confidence -->
    <div class="question-box">
      <p class="question-text">Har regionen endret klasse?</p>
      <div class="radio-group">
        <label>
          <input type="radio" bind:group={change_response} value="Change" required />
          Ja, regionen endret klasse?
        </label>
        <label>
          <input type="radio" bind:group={change_response} value="No Change" required />
          Nei, regionen forble uendret
        </label>
      </div>
      <p class="confidence-question">Hvor sikker er du på ditt svar?</p>
      <div class="confidence-options">
        <label class="option">
          <input type="radio" bind:group={change_confidence} value="5" required />
          <span class="option-text">Veldig sikker</span>
        </label>
        <label class="option">
          <input type="radio" bind:group={change_confidence} value="4" required />
          <span class="option-text">Noe sikker</span>
        </label>
        <label class="option">
          <input type="radio" bind:group={change_confidence} value="3" required />
          <span class="option-text">Nøytral</span>
        </label>
        <label class="option">
          <input type="radio" bind:group={change_confidence} value="2" required />
          <span class="option-text">Noe usikker</span>
        </label>
        <label class="option">
          <input type="radio" bind:group={change_confidence} value="1" required />
          <span class="option-text">Veldig usikker</span>
        </label>
      </div>
    </div>
    
    <!-- Question Box 2: Original Color & Confidence (only if answer is "Change") -->
    {#if change_response === "Change"}
      <div class="question-box">
        <p class="question-text">Hva var den opprinnelige klassen på regionen?</p>
        <div class="color-options">
          <label class="color-option {color_response === '#D3D3D3' ? 'selected' : ''}">
            <input type="radio" bind:group={color_response} value="#D3D3D3" required hidden />
            <span class="color-swatch" style="background-color: #D3D3D3;"></span>
            <span class="color-name">Lav</span>
          </label>
          <label class="color-option {color_response === '#A9A9A9' ? 'selected' : ''}">
            <input type="radio" bind:group={color_response} value="#A9A9A9" required hidden />
            <span class="color-swatch" style="background-color: #A9A9A9;"></span>
            <span class="color-name">Medium</span>
          </label>
          <label class="color-option {color_response === '#696969' ? 'selected' : ''}">
            <input type="radio" bind:group={color_response} value="#696969" required hidden />
            <span class="color-swatch" style="background-color: #696969;"></span>
            <span class="color-name">Høy</span>
          </label>
        </div>
        <p class="confidence-question">Hvor sikker er du på valget ditt?</p>
        <div class="confidence-options">
          <label class="option">
            <input type="radio" bind:group={color_confidence} value="5" required />
            <span class="option-text">Veldig sikker</span>
          </label>
          <label class="option">
            <input type="radio" bind:group={color_confidence} value="4" required />
            <span class="option-text">Noe sikker</span>
          </label>
          <label class="option">
            <input type="radio" bind:group={color_confidence} value="3" required />
            <span class="option-text">Nøytral</span>
          </label>
          <label class="option">
            <input type="radio" bind:group={color_confidence} value="2" required />
            <span class="option-text">Noe usikker</span>
          </label>
          <label class="option">
            <input type="radio" bind:group={color_confidence} value="1" required />
            <span class="option-text">Veldig usikker</span>
          </label>
        </div>
      </div>
    {/if}
    
    <div class="submit-container">
      <button class="submit-btn" on:click={submitAndNext} disabled={submitted}>
        {#if submitted && !demoMode}
          <svg class="check-icon" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="white" stroke-width="3" stroke-linecap="round" stroke-linejoin="round">
            <polyline points="20 6 9 17 4 12"></polyline>
          </svg>
        {:else}
          Neste
        {/if}
      </button>
    </div>
    {/if}
  </div>
</div>


<style>

  .survey-container {
    min-width: 440px;
    max-width: 480px;
    margin-top: 5px;
    background: white;
    padding: 15px;
    border-radius: 8px;
    box-shadow: 0px 4px 8px rgba(0, 0, 0, 0.1);
  }
  .color-options {
    display: flex;
    justify-content: center;
    gap: 20px;
    margin: 10px 0;
  }
  .color-option {
    cursor: pointer;
    text-align: center;
  }
  .color-option input {
    display: none;
  }
  .color-swatch {
    display: block;
    width: 50px;
    height: 30px;
    border: 2px solid transparent;
    margin: 0 auto 5px;
    border-radius: 6px;
    transition: border-color 0.3s ease;
  }
  .color-option:hover .color-swatch {
    border-color: blue;
  }
  .color-option.selected .color-swatch {
    border-color: blue;
  }
  .color-name {
    font-size: 14px;
  }
  .confidence-question {
    font-weight: bold;
    font-size: 14px;
    margin-bottom: 10px;
    margin-top: 8px;
  }
  .confidence-options {
    display: flex;
    gap: 15px;
    font-size: 14px;
    justify-content: center;
  }
  .option {
    display: flex;
    flex-direction: column;
    align-items: center;
  }
  .option-text {
    margin-top: 4px;
    font-size: 14px;
    text-align: center;
  }
  .content-wrapper {
    display: flex;
    flex-direction: row;
    gap: 20px;
    align-items: flex-start;
  }
  .progress-text {
    text-align: center;
    font-size: 14px;
    color: #666;
    margin-bottom: 8px;
  }
  .question-box {
    background: white;
    padding: 8px;
    padding-bottom: 20px;
    border-radius: 8px;
    border: 1px solid #ddd;
    margin-bottom: 10px;
    text-align: center;
  }
  .question-text {
    font-weight: bold;
    font-size: 16px;
    margin-bottom: 8px;
    margin-top: 8px;
    
  }
 


  .radio-group {
    display: flex;
    justify-content: center;
    flex-direction: row;
    gap: 15px;
    margin-bottom: 10px;
  }
  .radio-group label {
    display: block;
    margin: 5px 0;
    font-size: 14px;
  }
  .submit-container {
    text-align: center;
    margin-top: 20px;
  }
  .submit-btn {
    background: #007bff;
    color: white;
    border: none;
    padding: 10px 20px;
    border-radius: 5px;
    cursor: pointer;
    font-size: 14px;
  }
  .submit-btn:disabled {
    background: gray;
    cursor: not-allowed;
    padding: 8px 28px;
  }
  .check-icon {
    vertical-align: middle;
  }
</style>
