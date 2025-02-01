# Azure Open AI o3-mini streamlit app

## Introduction

- Testing o3mini with Azure Open AI
- using Streamlit app

## prerequisites

- Azure subscription
- Azure account
- Azure AI foundry account
- o3 mini in east US
- deploy the o3-mini in Azure AI foundry
- get the endpoint and key

## Steps

- idea here is to create a simple streamlit app
- Able to ask question or prompt
- Get response from o3 mini
- Display the response
- manage the chat history

### Code

```python
import datetime
import streamlit as st

import os  
import base64
from openai import AzureOpenAI  


endpoint = os.getenv("ENDPOINT_URL", "https://aifoundryname.openai.azure.com/")  
deployment = os.getenv("DEPLOYMENT_NAME", "o3-mini")  
subscription_key = os.getenv("AZURE_OPENAI_O3_KEY")  

# Initialize Azure OpenAI Service client with key-based authentication    
client = AzureOpenAI(  
    azure_endpoint=endpoint,  
    api_key=subscription_key,  
    api_version="2024-12-01-preview",
)
    
    
def processo3(query: str):
    #IMAGE_PATH = "YOUR_IMAGE_PATH"
    #encoded_image = base64.b64encode(open(IMAGE_PATH, 'rb').read()).decode('ascii')
    returntxt = ""

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
                    "text": f"""{query}"""
                }
            ]
        }
    ] 
        
    # Include speech result if speech is enabled  
    messages = chat_prompt  
        
    # Generate the completion  
    completion = client.chat.completions.create(  
        model=deployment,
        messages=messages,
        max_completion_tokens=100000,
        stop=None,  
        stream=False
    )

    # print(completion.to_json())  

    #print(completion)

    # Corrected access method
    #print(completion.choices[0].message.content)

    returntxt = completion.choices[0].message.content

    returntxt = returntxt + " \nCompletion Token: " + str(completion.usage.completion_tokens) + "\n"
    returntxt = returntxt + " Prompt Token: " + str(completion.usage.prompt_tokens) + "\n"
    returntxt = returntxt + " Total Token: " + str(completion.usage.total_tokens) + "\n"


    return returntxt

# Initialize chat history in session state
if "chat_history" not in st.session_state:
    st.session_state.chat_history = []

def showo3():
    st.header("o3-mini Test")

    
    if prompt := st.chat_input("describe quantum computing in detail and create a new design?", key="chat1"):
        # Call the extractproductinfo function
        #st.write("Searching for the query: ", prompt)
        st.chat_message("user").markdown(prompt, unsafe_allow_html=True)
        st.session_state.chat_history.append({"role": "user", "message": prompt})
        starttime = datetime.datetime.now()
        rfttopics = processo3(prompt)
        endtime = datetime.datetime.now()

        #st.markdown(f"Time taken to process: {endtime - starttime}", unsafe_allow_html=True)
        rfttopics += f"\n Time taken to process: {endtime - starttime}"
        st.session_state.chat_history.append({"role": "assistant", "message": rfttopics})
        st.chat_message("assistant").markdown(rfttopics, unsafe_allow_html=True)

        # Keep only the last 10 messages
        if len(st.session_state.chat_history) > 20:  # 10 user + 10 assistant
            st.session_state.chat_history = st.session_state.chat_history[-20:]
if __name__ == "__main__":
    showo3()
```

- Run the code

```
streamlit run o3mini.py
```

- output 

![info](https://github.com/balakreshnan/Samples2025/blob/main/AOAI/images/o3mini-1.jpg 'RagChat')

```
Below is a detailed overview of quantum computing – its principles, components, challenges, and current technologies – followed by a conceptual proposal for a novel quantum computer design that blends innovative ideas from multiple approaches.

──────────────────────────────

UNDERSTANDING QUANTUM COMPUTING
A. Basic Principles

• Qubits versus Classical Bits:
Unlike classical bits that are either 0 or 1, quantum bits (qubits) can exist in a superposition – a blend of 0 and 1 – which allows a quantum computer to process a large amount of computational space in parallel.

• Superposition:
A qubit may reside in a combination of basis states (|0⟩ and |1⟩) simultaneously. This is analogous to having many possible states active at once, leading to exponential growth in state-space with each added qubit.

• Entanglement:
Qubits can become entangled, meaning they share a quantum correlation regardless of the physical distance between them. A measurement on one qubit instantaneously affects the state of the other, enabling quantum algorithms to perform computations more efficiently than classical counterparts for certain problems.

• Interference:
Quantum states can interfere with one another either constructively or destructively. Algorithms harness interference to amplify correct answers while canceling out wrong ones.

B. Core Quantum Algorithms and Applications

• Shor’s Algorithm:
Uses quantum parallelism to factorize numbers exponentially faster than classical algorithms—important for breaking certain cryptographic schemes.

• Grover’s Algorithm:
Provides a quadratic speed-up for unstructured search problems, useful in optimization and database search tasks.

• Other Applications:
Quantum simulation of physical systems, optimization problems, machine learning, cryptographic protocols like quantum key distribution, and more.

C. Hardware Realizations and Challenges

• Common Physical Realizations:

Superconducting Qubits: Utilize Josephson junctions cooled to cryogenic temperatures.
Trapped Ions: Use electromagnetic traps to suspend ions, manipulated by laser beams.
Topological Qubits: Rely on exotic quasiparticles (e.g., Majorana fermions) offering inherent error protection.
Photonic Qubits: Leverage properties of photons for room-temperature operations and communication-friendly schemes.
• Challenges:

Decoherence: The loss of quantum coherence due to interactions with the environment requires error-correction techniques.
Scalability: Engineering large arrays of qubits with stable entanglement remains a major hurdle.
Error Correction: Quantum error correction requires redundancy (many physical qubits representing one logical qubit), complicating hardware design.
────────────────────────────── 2. A NOVEL QUANTUM COMPUTER DESIGN: THE "QUANTUM CONVERGENCE MACHINE" (QCM)

Imagine a next-generation architecture that fuses ideas from superconducting qubits, photonic interconnects, and modular design principles to attain improved scalability and error resilience. Below is a high-level blueprint for the concept.

A. Design Philosophy and Objectives

• Integration Across Technologies:
Combine the fast manipulation and control offered by superconducting qubits with the connectivity and low-loss transmission of photonic links.
• Modular Scalability:
Build the quantum computer as a network of smaller nodes (“quantum modules”) that can be interconnected. Each module can perform localized quantum logic, while photonic channels link modules to build a larger entangled system. • Enhanced Error Mitigation:
Utilize both state-of-the-art physical qubit error reduction (through materials and design) and a layered error-correction scheme leveraging physical-to-logical qubit encoding.

B. Key Components of the QCM

Superconducting Qubit Islands
• Functionality:
Each module includes a “qubit island” comprising superconducting qubits arranged on an integrated circuit chip cooled with dilution refrigeration technology.
• Advantages:
Fast quantum gate operations and established control electronics.

Tunable Quantum-Dot Interfaces
• Functionality:
Surrounding the superconducting islands, quantum-dot arrays (or spin qubits in semiconductors) act as dynamic interfaces that can switch between classical simulation modes and quantum error-correcting states.
• Advantages:
The versatility of these interfaces offers additional avenues for entangling qubits and implementing error correction.

Integrated Photonic Interconnects
• Functionality:
Connect modules via on-chip or chip-to-chip optical links capable of carrying entangled photons.
• Advantages:
Photonic channels provide low-loss long-distance entanglement, enabling distributed quantum processing and potentially networking quantum computers.

Layered Quantum Control Architecture
• Quantum Control Layer:
A high-speed microwave control system specifically tailored for superconducting qubit operations.
• Photonic Control Layer:
Optical controllers manage the timing and entanglement distribution in photonic interconnects.
• Error Correction and Synchronization Controller:
A dedicated digital unit that implements surface or concatenated code protocols, balancing physical errors against logical qubit fidelity delay.

C. Operating Concept of the QCM

Local Processing:
Within each module, high-fidelity gates are applied to perform basic quantum operations and local error correction using robust superconducting qubit arrays.

Distributed Entanglement:
When a computation limits local resources, the photonic interconnects create high-fidelity entangled links between modules allowing distributed computing over the entire system.

Hybrid Error Correction:
Through the tunable interfaces, the design dynamically switches between local error-correcting codes (optimized for superconducting systems) and network-tailored error detection across modules. This dual-level approach may reduce overhead compared to deploying a single error-correction scheme across an entire large-scale quantum computer.

Scalability through Modularity:
New modules can be added as “nodes” in a quantum network. Each module undergoes independent calibration yet conforms to the standardized photonic interconnect protocol, allowing for robust scaling without necessitating entirely new hardware designs.

D. Potential Advantages and Future Research Directions

• Enhanced Connectivity:
Photonic interconnects allow separate quantum modules to communicate without the complications of microwave crosstalk found in densely packed superconducting systems.

• Flexibility and Resilience:
Modular design permits isolation and repair of faulty nodes while the overall network continues operation—a desirable trait for real-world quantum computing systems.

• Incremental Implementation:
The design supports a roadmap in which early iterations involve a few interconnected modules. Over time, success in scaling error correction and photonic communication can lead to a full-fledged quantum supercomputer.

• Research Path:
Work would be needed to optimize coupling between superconducting qubits and quantum dots, to design low-loss, high-fidelity photonic channels, and to develop integrated control electronics that can handle the delicate timing and calibration required across communication layers.

────────────────────────────── 3. CONCLUSION

Quantum computing, by harnessing the counterintuitive principles of superposition and entanglement, promises to solve certain types of problems exponentially faster than classical machines. The Quantum Convergence Machine (QCM) design outlined above is a conceptual framework that addresses scalability and connectivity—the two largest hurdles today—by blending state-of-the-art superconducting qubits with photonic interconnects and novel error-correction techniques.

While this design remains at the conceptual stage, it illustrates how a multi-disciplinary integration of quantum hardware technologies might pave the way for practical, large-scale quantum computing in the future. As research continues in material science, photonics, and quantum error correction, concepts like the QCM may well inspire next-generation quantum architectures. Completion Token: 2105 Prompt Token: 32 Total Token: 2137

Time taken to process: 0:00:15.731987
```

- done
- Have fun testing.