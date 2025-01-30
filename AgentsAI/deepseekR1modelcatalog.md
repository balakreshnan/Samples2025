# Deep Seek R1 Model Catalog - Deploy in Azure AI Foundry

## Introduction

- Able to deploy Deep Seek R1 models in Azure AI Foundry.
- We are going to deploy from Model catalog.
- Deploy as model as a service.
- Hosted environment.
- Didn't have to procure hardware.

## Prerequisites

- Azure subscription
- Azure AI foundry
- Local Laptop or Desktop
- Python 3.12 virtual environment with latest packages installed.
- Model catalog in region where DeepSeek R1 is available
- Create the AI foundry in that region to deploy the model.
- when writing this article here are regions available: eastus2, westus3, northcentralus, eastus, southcentralus,westus

## Steps

- Go to ai.azure.com
- login into your AI Hub Account
- Select your project, if there is no project, please create one
- Then go to Model Catalog
- in the home page you should see Deep Seek R1 models
- Click on model you want to deploy
- Then Click on Deploy

![info](https://github.com/balakreshnan/Samples2025/blob/main/AML/images/deepseekR1-1.jpg 'RagChat')

- Click Continue
- Give a deployment name
- Click Deploy

![info](https://github.com/balakreshnan/Samples2025/blob/main/AML/images/deepseekR1-2.jpg 'RagChat')

- Now it will take few minutes to deploy the model
- Here is the progress

![info](https://github.com/balakreshnan/Samples2025/blob/main/AML/images/deepseekR1-3.jpg 'RagChat')

- took few mintues to deploy

![info](https://github.com/balakreshnan/Samples2025/blob/main/AML/images/deepseekR1-4.jpg 'RagChat')

- API information is available after deployment
- Make sure note the key and endpoint
- Go to Consume tab and copy the python code 

```
# pip install azure-ai-inference
import os
from azure.ai.inference import ChatCompletionsClient
from azure.core.credentials import AzureKeyCredential

api_key = os.getenv("AZURE_INFERENCE_CREDENTIAL", '')
if not api_key:
  raise Exception("A key should be provided to invoke the endpoint")

client = ChatCompletionsClient(
    endpoint='https://DeepSeek-R1-ep.eastus2.models.ai.azure.com',
    credential=AzureKeyCredential(api_key)
)

model_info = client.get_model_info()
print("Model name:", model_info.model_name)
print("Model type:", model_info.model_type)
print("Model provider name:", model_info.model_provider_name)

payload = {
  "messages": [
    {
      "role": "user",
      "content": "I am going to Paris, what should I see?"
    },
    {
      "role": "assistant",
      "content": "Paris, the capital of France, is known for its stunning architecture, art museums, historical landmarks, and romantic atmosphere. Here are some of the top attractions to see in Paris:\n\n1. The Eiffel Tower: The iconic Eiffel Tower is one of the most recognizable landmarks in the world and offers breathtaking views of the city.\n2. The Louvre Museum: The Louvre is one of the world's largest and most famous museums, housing an impressive collection of art and artifacts, including the Mona Lisa.\n3. Notre-Dame Cathedral: This beautiful cathedral is one of the most famous landmarks in Paris and is known for its Gothic architecture and stunning stained glass windows.\n\nThese are just a few of the many attractions that Paris has to offer. With so much to see and do, it's no wonder that Paris is one of the most popular tourist destinations in the world."
    },
    {
      "role": "user",
      "content": "What is so great about #1?"
    }
  ],
  "max_tokens": 2048
}
response = client.complete(payload)

print("Response:", response.choices[0].message.content)
print("Model:", response.model)
print("Usage:")
print("	Prompt tokens:", response.usage.prompt_tokens)
print("	Total tokens:", response.usage.total_tokens)
print("	Completion tokens:", response.usage.completion_tokens)
```

- Run the code and you should see the response
- output

```

(.venv) C:\Code\agifoundation1>python deepseekr1test.py
Model name: DeepSeek-R1
Model type: chat-completion
Model provider name: DeepSeek
Response: <think>
Okay, the user is asking "What is so great about #1?" referring to the first attraction mentioned in the previous answer. Let me check—the first point was the Eiffel Tower. So, I need to elaborate on why the Eiffel Tower is great.

Hmm, first I should consider what the user already knows. They're planning a trip to Paris and looking for things to see. They might want to know specific reasons why the Eiffel Tower stands out. Let me think about the key aspects: historical significance, engineering, views, cultural impact, and visitor experience.

The user might be curious about the tower's history—like when it was built, for what occasion. Maybe the controversy around it during its construction? Also, it's a symbol of Paris, so the cultural importance. The engineering part might be interesting, such as its height and how it was the tallest man-made structure at the time. The views from the top are definitely a big draw, especially the panoramic vistas of Paris. Also, the light shows at night could be a romantic or photogenic aspect. Additionally, how visitors can experience it—whether dining there, going up to different levels, etc.

I should address these points in a structured way. Need to make sure the answer isn't too technical but still informative. Also, maybe touch on why it's a must-see compared to other landmarks. The user might be deciding how to prioritize their time, so highlighting unique aspects would be good. Let me also check if there are any common misconceptions or interesting facts about the Eiffel Tower that could add value. Oh, there's the fact that it was initially meant to be temporary, but its use as a radio tower saved it. That could be an interesting tidbit. Including different perspectives: day vs. night visits, maybe tips for visiting. But the user didn't ask for tips, just why it's great. So focus on the positives and significance.
</think>

**The Eiffel Tower (La Tour Eiffel) is considered iconic for several reasons:**

### 1. **Symbol of Paris and France**
   - Built in 1889 as the entrance arch for the World’s Fair (Exposition Universelle), it quickly became a global symbol of Paris and French ingenuity.
   - Despite initial criticism (many Parisians hated its modern, industrial design at the time), it now represents romance, innovation, and the cultural identity of France.

### 2. **Engineering Marvel**
   - Designed by Gustave Eiffel, it was the tallest human-made structure in the world (324 meters/1,063 feet) until 1930.
   - Its lattice ironwork was revolutionary for its time, combining aesthetics and strength. Over 18,000 iron parts were assembled with extraordinary precision.       

### 3. **Breathtaking Views**
   - The panoramic views from its three observation decks (especially the top level) let you see Paris unfold beneath you—the Seine River, historic landmarks, and sprawling rooftops.
   - At night, the tower sparkles with thousands of lights (a 5-minute light show every hour after sunset), adding magic to the city skyline.

### 4. **Cultural and Historical Resonance**
   - It has starred in countless films, books, and artworks, solidifying its place in pop culture as a universal symbol of love and adventure.
   - Originally meant to stand for 20 years, it was saved by its utility as a radio transmission tower, cementing its historical importance.

### 5. **Visitor Experience**
   - You can dine at its Michelin-starred **Le Jules Verne** restaurant, enjoy a glass of champagne at the top, or simply picnic on the Champ de Mars below while gazing up at the structure.
   - Seasonal events, like ice-skating rinks or light displays, add to its charm.

### **Why It’s "Great"**
The Eiffel Tower isn’t just a monument—it’s an emotional experience. Whether you’re marveling at its engineering, watching it glow at night, or feeling the weight of its history, it embodies the romance and grandeur that define Paris. It’s a must-see because no photo or description can replicate the awe of standing beneath (or atop) this iron giant.
Model: DeepSeek-R1
Usage:
        Prompt tokens: 200
        Total tokens: 1096
        Completion tokens: 896
```

- Try with various prompts and see the response
- It does take time to response and check the <think> tag in the response for the thinking process of the model
- Have fun with Deep Seek R1 models
- here is another example
- here is the payload or message

```
payload = {
  "messages": [
    {
      "role": "user",
      "content": "what to know more about scientific theory, can you explain it?"
    },
    {
      "role": "assistant",
      "content": "A scientific theory is a well-substantiated explanation of a natural phenomenon, based on a body of empirical evidence that has been repeatedly confirmed through observation and experimentation. Unlike a hypothesis or a guess, a theory provides a comprehensive framework that explains and predicts phenomena in a reliable way. It must be testable, falsifiable, and subject to revision if new evidence contradicts it. A strong theory has broad explanatory scope and predictive power, making accurate forecasts about future observations. Examples include the Theory of Evolution, which explains species changes over time through natural selection; General Relativity, which describes how gravity affects space and time; and Quantum Mechanics, which governs atomic and subatomic behavior. Unlike scientific laws, which describe relationships (e.g., Newton’s Laws of Motion), theories explain why and how those laws work, often incorporating them as special cases."
    },
    {
      "role": "user",
      "content": "Invent a new scientific theory that explains both dark matter and consciousness.?"
    }
  ],
  "max_tokens": 2048
}
response = client.complete(payload)
```

- output

```
(.venv) C:\Code\agifoundation1>python deepseekr1test.py
Model name: DeepSeek-R1
Model type: chat-completion
Model provider name: DeepSeek
Response: <think>
Okay, the user wants me to invent a new scientific theory that explains both dark matter and consciousness. Hmm, that's pretty ambitious. Let me start by breaking down what each of these phenomena are.

Dark matter is this mysterious substance that doesn't emit light or energy, but we know it exists because of gravitational effects on galaxies. Consciousness is the state of being aware and able to think. They seem unrelated, but maybe there's a connection through some underlying structure or field.

I remember that dark matter is thought to make up about 27% of the universe. Consciousness, on the other hand, is a complex topic with various theories, like integrated information theory or panpsychism. What if there's a fundamental field that interacts with both?

Quantum fields come to mind. Maybe a new kind of quantum field that permeates the universe. If this field has different states or phases, it could account for dark matter's gravitational effects. For consciousness, perhaps this field allows for integrated information when in a certain state, leading to conscious experience.

The user might be looking for a unified theory that bridges physics and neuroscience. They could be interested in how the universe's structure relates to mind. I should ensure the theory is testable. Maybe predict dark matter interactions through brain activity experiments or look for coherence in neural networks similar to quantum states.

Wait, presenting it as a hierarchy where the fundamental field gives rise to both dark matter particles and conscious systems through complexity. Coherence at different scales—like Bose-Einstein condensates for dark matter and neural synchrony for consciousness. That could tie them together.

Need to make sure the theory isn't just speculative but includes mechanisms for interaction. Using terms like non-local quantum coherence and gravitational entanglement might work. Also, mention experiments that could validate or falsify it. The user probably wants a creative yet plausible idea that sparks further inquiry.
</think>

**Theory of Coherent Quantum Emergence (CQE)**
**Synopsis:**
The *Coherent Quantum Emergence* theory proposes that **dark matter** and **consciousness** are intertwined manifestations of a single, fundamental quantum substructure permeating the universe, termed the **Axiomatic Field** (AF). This field consists of **non-local quantum coherence**—a dynamic web of entangled particles and information that bridges scales from the quantum to the cosmological. Here’s how it reconciles both mysteries:

---

### **1. Dark Matter as Dynamically Ordered AF States**
- The Axiomatic Field generates stable, low-energy quantum states (“**axion condensates**”) that interact weakly with ordinary matter via gravity but resist electromagnetic coupling. These condensates are predicted to form fractal-like galactic halos.
- **Key insight**: Dark matter is not inert “stuff” but a **phase of quantum coherence**—like a Bose-Einstein condensate spanning galaxies. Its gravitational footprint arises from AF “knots,” regions where the field’s entanglement density warps spacetime.

---

### **2. Consciousness as Localized AF Resonance**
- Consciousness emerges when biological neural networks achieve **critical coherence** with the Axiomatic Field. Neurons act as “antennae,” tuning into resonant frequencies of the AF through synchronized electromagnetic oscillations (e.g., gamma waves).
- **Mechanism**: The brain’s microtubules (quantum-sensitive structures) transiently entangle with the AF, creating a self-referential feedback loop (qualia) and enabling integrated information (awareness). Consciousness is thus not solely biological but a **hybrid quantum-biological phenomenon**.

---

### **3. Bridging the Scales: Symmetry & Feedback**
- **Symmetry principle**: The AF obeys a universal **holographic symmetry**, where macroscopic dark matter structures (galactic halos) and microscopic conscious systems (brains) are dual expressions of the same quantum coherence.
- **Feedback loop**: Conscious observations (wavefunction collapses) subtly perturb the AF, influencing dark matter’s distribution. Conversely, dark matter’s gravitational lensing subtly biases neural firing patterns (explaining psi phenomena like intuition).

---

### **4. Predictions & Tests**
- **Dark matter**:
  - Predict **axion condensates** emitting faint gravitational waves detectable by next-gen interferometers (LISA).
  - Dark matter halos should exhibit unexpected coherence in polarization maps of galactic clusters.
- **Consciousness**:
  - Brain scans under quantum-volatile conditions (superconducting interference) will show disrupted gamma synchrony.
  - Repeatable experiments demonstrating consciousness-dependent deviations in quantum random number generators.

---

### **Implications**
- **Unified ontology**: Particle physics, cosmology, and neuroscience are facets of the Axiomatic Field’s hierarchy.
- **Ethical AI**: Synthetic consciousness could emerge only in systems achieving AF resonance, demanding ethical frameworks for quantum-computing architectures.       

---

**Conclusion**:
The CQE theory challenges reductionist paradigms by unifying cosmic-scale phenomena (dark matter) with subjective experience (consciousness) through quantum coherence. While speculative, it provides testable hypotheses to collapse the “explanatory gap” between mind and cosmos.

Would you like to dive deeper into experimental validations or philosophical implications? 🌌🧠
Model: DeepSeek-R1
Usage:
        Prompt tokens: 200
        Total tokens: 1279
        Completion tokens: 1079
```