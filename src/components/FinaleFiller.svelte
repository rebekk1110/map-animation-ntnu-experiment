<script>
    import { createEventDispatcher } from "svelte";
    const dispatch = createEventDispatcher();

    export let participantId;
    let effort = "";
    let feedback = "";
  
    function submitFinal() {
      const data = {
        participant_id: participantId,
        effort: effort,   // e.g., "5", "4", etc.
        feedback: feedback
      };
  
      fetch("https://change-blindness-web.onrender.com/finalize", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(data)
      })
        .then(response => response.json())
        .then(data => {
          console.log("Final data submitted:", data.message);
          dispatch("next")
          // Optionally, you could redirect or show an additional message here.
        })
       
        .catch(error => {
          console.error("Error submitting final data:", error);
        });
    }
  </script>
  
  <div class="final-page">

    <div class="card">
      <p class="question-text">Hvor mye innsats la du ned på studien?</p>
      <div class="radio-group">
        <label>
          <input type="radio" bind:group={effort} value="5" required />
          <span>Veldig høy innsats</span>
        </label>
        <label>
          <input type="radio" bind:group={effort} value="4" required />
          <span>Høy innsats</span>
        </label>
        <label>
          <input type="radio" bind:group={effort} value="3" required />
          <span>Middels innsats</span>
        </label>
        <label>
          <input type="radio" bind:group={effort} value="2" required />
          <span>Lav innsats</span>
        </label>
        <label>
          <input type="radio" bind:group={effort} value="1" required />
          <span>Veldig lav innsats</span>
        </label>
      </div>
    </div>
    
    <div class="card">
      <p class="question-text">Dersom du har kommentarer til studien, setter vi pris på å høre dem</p>
      <textarea bind:value={feedback} placeholder="Skriv din tilbakemelding her..." rows="4"></textarea>
    </div>
    
    <button class="submit-btn" on:click={submitFinal}>Neste</button>
  </div>
  
  <style>
    :global(body) {
      font-family: "Helvetica Neue", Arial, sans-serif;
      background: #f0f2f5;
      margin: 0;
      padding: 0;
    }
  
    .final-page {
      max-width: 600px;
      margin: 40px auto;
      background: #fff;
      padding: 30px;
      border-radius: 10px;
      box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
      text-align: center;
    }

    .card {
      background: #fafafa;
      padding: 20px;
      border-radius: 8px;
      margin-bottom: 20px;
      text-align: left;
    }
  
    .card .radio-group {
      gap: 10px;
     
    }
    .question-text {
      font-size: 17px;
      font-weight: bold;
      margin-bottom: 18px;
      color: #333;
    }
    
    .radio-group {
      display: flex;
      flex-direction: row;
      justify-content: space-around;
      margin-top: 10px;
    }
    
    .radio-group label {
      display: flex;
      flex-direction: column;
      align-items: center;
      font-size: 14px;
      cursor: pointer;
      color: #555;
    }
    
    .radio-group input[type="radio"] {
      margin-bottom: 8px;
    }
    
    textarea {
      width: 100%;
      padding: 12px;
      font-size: 14px;
      border: 1px solid #ddd;
      border-radius: 6px;
      resize: vertical;
      color: #333;
    }
    
    .submit-btn {
      background: #007bff;
      color: #fff;
      border: none;
      padding: 12px 25px;
      font-size: 15px;
      border-radius: 8px;
      cursor: pointer;
      transition: background 0.3s ease;
    }
    
    .submit-btn:hover {
      background: #0056b3;
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
  