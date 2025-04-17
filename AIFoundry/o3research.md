# Azure Open AI o3 models in Azure AI Foundry

## Introduction

- Azure Open AI o3 models are now available in Azure AI Foundry.
- Use the o3 model to create source code and run it in Azure AI Foundry.
- Sample to run using Visual Studio Code and Azure AI Foundry.
- This is just a test to show case how to consume the o3 model in Azure AI Foundry.

## Pre-requisites

- Azure Subscription
- Azure AI Foundry
- Azure Open AI o3 model
- Visual Studio Code
- python 3.12 or later
- Install Open AI SDK for Python

## Code

- Go to Azure AI Foundry, deploy the model from Model catalog.
- Set the environment variable for Azure Open AI o3 model
- Make sure Azure open ai endpoint points to Azure AI Foundry URL

- here is the code to run the o4 mini model in Azure AI Foundry.

```
from openai import AzureOpenAI
import time
import os

from dotenv import load_dotenv
load_dotenv()

client = AzureOpenAI(
  azure_endpoint = os.getenv("AZURE_OPENAI_ENDPOINT"), 
  api_key=os.getenv("AZURE_OPENAI_API_KEY"),  
  api_version="2025-01-01-preview",
)

model_name = os.getenv("AZURE_OPENAI_DEPLOYMENT")

def get_o3_response(prompt, max_tokens=150):
    returntxt = ""
    try:
        

        #Prepare the chat prompt 
        chat_prompt = [
            {
                "role": "developer",
                "content": [
                    {
                        "type": "text",
                        "text": "You are an AI assistant that helps people find information."
                    }
                ]
            },
            {
                "role": "user",
                "content": [
                    {
                        "type": "text",
                        "text": f"""{prompt}"""
                    }
                ]
            }
        ] 
            
        # Include speech result if speech is enabled  
        messages = chat_prompt  
            
        start_time = time.time()
        # Generate the completion  
        completion = client.chat.completions.create(  
            model="o3",
            messages=messages,
            max_completion_tokens=100000,
            stop=None,  
            stream=False 
        )

        # print(completion.to_json())  

        # print(completion)
        end_time = time.time()
        time_taken = end_time - start_time

        # Corrected access method
        #print(completion.choices[0].message.content)

        returntxt = completion.choices[0].message.content

        returntxt = returntxt + " \nCompletion Token: " + str(completion.usage.completion_tokens) + "\n"
        returntxt = returntxt + " Prompt Token: " + str(completion.usage.prompt_tokens) + "\n"
        returntxt = returntxt + " Total Token: " + str(completion.usage.total_tokens) + "\n"
        print(f"Time taken: {time_taken:.3f} seconds")
        returntxt = returntxt + f"Time taken: {time_taken:.3f} seconds"
        return returntxt
    except Exception as e:
        print("Error obtaining response:", e)
        returntxt = str(e)
        return None
    


if __name__ == "__main__":
    # user_prompt = "Explain the significance of the Pythagorean theorem in modern mathematics."
    user_prompt = "What are the underlying causes of the shifting North Pole, and what are the associated benefits and potential risks of this phenomenon?"
    output = get_o3_response(user_prompt)
    if output:
        print("O3 Response:")
        print(output)
    else:
        print("No response received.")
```

- run the code using the command line or Visual Studio Code.
- Run the app
- output

```
Time taken: 157.180 seconds
O3 Response:
When people talk about the “North‑Pole drift” they usually mix together two different things:

1. Magnetic North Pole – the spot where Earth’s magnetic field points straight down.  
2. Rotational (geographic) North Pole – the end of Earth’s spin axis, the point that defines latitude 90 ° N.

Both are moving, but for completely different reasons and at very different speeds. Understanding the distinction is the key to understanding causes, benefits and risks.

────────────────────────────────
1. Magnetic North Pole
────────────────────────────────
Observed change  
• 1831 – 1990s: crept across the Canadian Arctic at 10–15 km / yr.  
• 1990s – present: sped up to ~50 km / yr toward Siberia; now north of 86 ° N.

Underlying causes  
• Fluid motion in the liquid outer core. Convection, affected by Earth’s rotation and heat loss, constantly rearranges the electrically‑conducting iron alloy.  
• These motions continually re‑shape the geodynamo, shifting “flux lobes” of strong and weak field at the core–mantle boundary.  
• Interaction of those lobes with each other and with outer‑core flow accelerates or decelerates the surface drift.  
• No link to climate; the driver is ~3 000 km below us.

Potential benefits  
• A moving pole forces frequent updates to the World Magnetic Model (WMM). Each update improves our picture of core dynamics and helps test geodynamo theories.  
• Refining the field model improves magnetic surveying for mineral, oil and archaeological exploration.  
• Changes in the auroral oval follow the pole; some Arctic communities gain more frequent aurora, boosting tourism.  
• The advance warnings and revised charts raise general navigation safety.

Potential risks  
• Compass error grows fastest at high latitudes; pilots, ships, drones, surveyors and even pipeline‑laying operations must apply new magnetic declination values or risk heading errors.  
• Airport runways designated by magnetic bearing (e.g., “36R”) need renumbering when the shift exceeds 5°.  
• A migrating pole is usually accompanied by regional field weakening (e.g., the South Atlantic Anomaly). Weaker field lets more high‑energy particles penetrate:  
  – Increased radiation dose to high‑altitude flights and astronauts.  
  – Higher satellite anomaly rates, shorter satellite lifetimes, worse space‑weather interference with radio links.  
• If the drift heralds a long‑term geomagnetic reversal (timescale ≥1 000 yr), field strength could drop to 10–20 % of its present value for centuries, magnifying all of the above hazards and stressing power grids and long pipelines during geomagnetic storms.

────────────────────────────────
2. Rotational (Geographic) North Pole
────────────────────────────────
Observed change  
• The spin axis wobbles naturally (Chandler wobble, ~9‑m positional circle) and drifts ~10 cm / yr across the Arctic Ocean.  
• Since the 1990s the drift direction changed from Canada‑ward to Russia‑ward and sped up to ~17 cm / yr.

Underlying causes  
• Mass redistribution on and within Earth alters the planet’s moment of inertia; the axis shifts so total angular momentum stays constant (“true polar wander” on modern time scales). The largest contributors since 1990 are:  
  – Rapid loss of ice from Greenland, West Antarctica and mountain glaciers.  
  – Ground‑water pumping and reservoir impoundment that moves water from aquifers to the oceans.  
  – Ongoing mantle rebound under formerly glaciated regions (glacial isostatic adjustment).  
  – Large earthquakes and volcanic events (small but measurable).  
• These processes are ultimately linked to climate change and tectonics, not to core dynamics.

Potential benefits  
• Tracking the axis drift provides an independent, sensitive indicator of modern mass‑balance changes—essential for closing the global sea‑level budget and validating climate models.  
• Improved understanding of Earth’s rotation helps maintain precise timekeeping (UT1), satellite orbits and deep‑space tracking.  
• Small pole shifts slightly redistribute sea‑surface height, which marginally relieves relative sea‑level rise in a few regions.

Potential risks  
• For most of society, a 10–20 cm / yr pole drift is imperceptible. The main issues are technical:  
  – Requires continual refinement of the International Terrestrial Reference Frame used for GPS, GNSS, and Earth‑observation satellites.  
  – If uncorrected, errors could creep into ultra‑precise geodesy, autonomous farming, earthquake‑strain monitoring and ballistic‑missile guidance.  
• On longer (10^5–10^6 yr) time‑scales, large true‑polar‑wander episodes could shift continents through different climatic zones, but nothing that large is happening now.

────────────────────────────────
3. Putting the numbers in context
────────────────────────────────
• Magnetic drift today ≈ 50 km / yr – large for navigation, negligible for climate.  
• Rotational drift today ≈ 0.17 m / yr – negligible for navigation, critical for precision geodesy.  

────────────────────────────────
4. Key take‑aways
────────────────────────────────
1. The fast‑moving Magnetic North Pole is controlled by liquid‑core dynamics; its main practical impacts are on navigation, satellite radiation environment and, if a prolonged weakening occurs, on technological infrastructure.  
2. The slow drift of the geographic pole is primarily a fingerprint of modern climate‑driven mass redistribution and affects only the most precise positioning and timing systems.  
3. Benefits are mostly scientific (better Earth‑system monitoring) and minor economic (updated charts, polar tourism); risks range from modest (runway renumbering) to potentially serious during extreme geomagnetic storms or a future field reversal. 
Completion Token: 2148
 Prompt Token: 45
 Total Token: 2193
Time taken: 157.180 seconds
```

- More research use case to follow.