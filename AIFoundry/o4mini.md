# Azure Open AI 4.1 models in Azure AI Foundry

## Introduction

- Azure Open AI o4 mini models are now available in Azure AI Foundry.
- Use the o4 mini model to create source code and run it in Azure AI Foundry.
- Sample to run using Visual Studio Code and Azure AI Foundry.
- This is just a test to show case how to consume the o4 mini model in Azure AI Foundry.

## Pre-requisites

- Azure Subscription
- Azure AI Foundry
- Azure Open AI o4 mini model
- Visual Studio Code
- python 3.12 or later
- Install Open AI SDK for Python

## Code

- Go to Azure AI Foundry, deploy the model from Model catalog.
- Set the environment variable for Azure Open AI o4 mini model
- Make sure Azure open ai endpoint points to Azure AI Foundry URL

- here is the code to run the o4 mini model in Azure AI Foundry.

```python
import os
from openai import AzureOpenAI

endpoint = os.getenv("AZURE_OPENAI_ENDPOINT")
model_name = "o4-mini"
deployment = "o4-mini"

subscription_key = os.getenv("AZURE_OPENAI_API_KEY")
api_version = "2024-12-01-preview"

client = AzureOpenAI(
    api_version=api_version,
    azure_endpoint=endpoint,
    api_key=subscription_key,
)

def main(query: str) -> str:
    returnstr = ""

    response = client.chat.completions.create(
        messages=[
            {
                "role": "system",
                "content": "You are a helpful assistant.",
            },
            {
                "role": "user",
                "content": f"{query}",
            }
        ],
        max_completion_tokens=100000,
        model=deployment
    )

    # print(response.choices[0].message.content)
    returnstr = response.choices[0].message.content
    return returnstr

if __name__ == "__main__":
    query = "What are the underlying causes of the shifting North Pole, and what are the associated benefits and potential risks of this phenomenon?"
    result = main(query)
    print(f"Result: {result}")
```

- run the code using the command line or Visual Studio Code.
- Run the app

- output

```
Result: Earth’s “North Pole” can be said to wander in two quite different senses: (1) the geographic rotation‐axis pole (true or “spin” pole), which currently drifts by a few centimeters each year, and (2) the geomagnetic North Pole, which has in recent decades been racing across the Canadian Arctic toward Siberia at up to 50 km/yr.  Below is an outline of what drives each, and what benefits and risks follow.

1.  Geographic “True” Pole Wandering  
   Causes  
   •  Mass redistribution in the solid Earth and on its surface.  In particular:  
     –  Glacial isostatic adjustment: as ice sheets from the last Ice Age have continued to melt, the crust rebounds and the load shifts.  
     –  Ongoing ice‐melt today (Greenland, Antarctica) and redistribution of that water into the oceans.  
     –  Large‐scale groundwater depletion (pumping for agriculture and cities).  
     –  Mantle convection and plate tectonic reorganization on long time scales.  
   •  Fluid motions in the liquid core and in the oceans also make small contributions.  

   Benefits  
   •  The very fact that we can measure this wandering with millimeter precision is a triumph of modern geodesy and helps us refine models of Earth’s interior, sea‐level rise, and glacial history.  
   •  Improved Earth‐orientation parameters feed into more accurate GPS positioning, satellite tracking, meteorology, and astronomy.  

   Risks/Drawbacks  
   •  If not continually monitored and updated, even a few centimeters of pole shift can introduce biases into high-precision surveying, oil‐and‐gas exploration, and other geodetic applications.  
   •  Small corrections must constantly be made to navigation systems (such as inertial navigation in aircraft and ships), though such updates are routine.  

2.  Geomagnetic North‐Pole Drift  
   Causes  
   •  Fluid‐dynamical convection of molten iron alloys in Earth’s outer core.  Changes in flow patterns alter the geomagnetic field, causing the dipole axis to migrate.  
   •  Interaction between the dipole field and smaller‐scale field components (so‐called “flux lobes”) that appear, move, and decay under the core‐mantle boundary.  

   Benefits  
   •  Offers a window into deep‐Earth dynamics.  Continuous satellite missions (e.g. Swarm) and ground observatories map these changes, improving our understanding of the geodynamo.  
   •  Data on field strength and geometry feed into space‐weather forecasting (which protects satellites and power grids).  

   Risks/Drawbacks  
   •  Compasses and navigation systems that rely on a fixed magnetic pole must be recalibrated more frequently—important for aviation, maritime operations, and even some smartphone apps.  
   •  If the geomagnetic dipole weakens significantly (as it has over the past 150 years), Earth’s natural shield against cosmic rays and charged solar particles is reduced, potentially increasing radiation exposure for satellites, high‐altitude aircraft crews, and, during excursions into polar regions, airline passengers.  
   •  Rapid or large‐scale shifts can fuel public concern and geopolitical disputes over new shipping lanes or potential resource claims in the Arctic, though the pole’s motion itself does not grant any legal territory.  

Summary  
– The geographic pole wanders only a few centimeters per year, driven mainly by mass shifts in the crust, fluids, and ice, and must be tracked to keep high‐precision positioning and navigation accurate.  
– The geomagnetic pole can move tens of kilometers per year, driven by the turbulent outer core; it commands attention from navigation services, space‐weather forecasters, and anyone relying on magnetic compasses.  
– In both cases the “risks” are almost entirely technological and logistical, offset by large scientific and practical benefits in Earth‐system monitoring, navigation, and hazard mitigation.
```

- More to come
- Seems little slow, but it is working.