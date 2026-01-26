### Some Use cases
1. **Create Content** - `generate blog post for ideas on future of AI`
2. **Translate Text** - `translate this text to english` "sample text"
3. **Create an image in a specific style** - `turn this imaeg into pencil style sketch`
   - `create an image that shows the data hungry nate of large generative models, no words` 
5. **Describe an Image** - `describe the objects and color in this image` - (upload image)
6. **Visualize content** - `create a visual diagram to outline given steps`
7. **Proofread** - `Proofread this email to find any mistakes:` line break _followed by email content as paste_
8. **Summarize text** - `summarize this research paper` - _attach document probably PDF or word etc_ OR _you can also provide a http link url_
   - `summarize this article` https://<Your_Url_Name
     - Further followup example `give me  rankings of content ` - etc

#### Types of Prompting
* **Zero-shot Prompting**
  * Examples
    * `Trim the text to be more succinct` - _line break followed by text to be shorterned_
    * Can be more shortened if you are specific 
* **Few-shot Prompting**
  * Give examples of input and output text. Give one more example (_All in the same prompt_).   
* **Chain-of-thought Prompting**
  * Explain the thought process by making as much descriptive as much possible.  
* **Prompt Chaining**
  * Instead of doing 1 Big task break it into smaller prompts and use that output as input for next task. Consider like giving task to an employee all at once.  
* **Reverse Prompting**
  *  Ask model for prompt by using few short prompting of giving a prompt with answer already
 
#### Refining Prompt 
* Ensure the level of specificity.
* Define a persona or style you expect (professional / business)
* Provide Context (or example)
* Output the format you want for the response.

Small change in query changes the output
<img width="1349" height="652" alt="image" src="https://github.com/user-attachments/assets/7edea636-283f-4976-ae5d-d9b5a04f8781" />

Verifying Outputs
* Humans in the loop ✔️
* Statistical Eval dataset ℹ️
* use LLM as judge ⚠️ (_treat it as an additional tool only_). Be upfront about that for example be clear like _this is an AI Generated text_
  * <img width="1352" height="756" alt="image" src="https://github.com/user-attachments/assets/7bec2815-875b-4c50-bfb4-1cd9cf26691e" />
 
#### Video Prompting
Tools example
* Google Veo
* OpenAI Sora
* Runway
* Invideo AI

Use a Prompt or article e.g `street inverview` etc

References: https://learning.oreilly.com/videos/prompt-engineering-basics/9780135496909 by Kate Harwood. 



