# Samples2025: Cutting-Edge AI Technology Demonstrations

A comprehensive collection of hands-on examples showcasing the latest AI technologies, models, and Azure AI services for 2025. This repository provides practical, real-world implementations across enterprise scenarios including manufacturing, healthcare, supply chain management, and more.

## 🌟 Repository Value & Context

This repository serves as a learning hub and reference implementation for:

- **Latest AI Models**: Demonstrations of cutting-edge models including GPT-4, o1, o3-mini, DeepSeek R1, Phi-4, Llama 3.2, and more
- **Azure AI Services Integration**: Comprehensive examples using Azure OpenAI, Azure AI Foundry, Azure ML, and Azure AI Studio
- **Enterprise AI Solutions**: Real-world scenarios for manufacturing compliance, supply chain optimization, healthcare planning, and production scheduling
- **Responsible AI Practices**: Implementation of RAI (Responsible AI) evaluation, governance, and compliance frameworks
- **Multi-Agent Systems**: Advanced agent orchestration using Microsoft Autogen and Azure AI agent frameworks
- **Code-First Approaches**: SDK-based implementations alongside UI-driven solutions

## 🚀 Key Technologies Featured

### AI Models & Frameworks
- **Large Language Models**: GPT-4, o1, o3-mini, DeepSeek R1, Phi-4, Llama 3.2, MobileLLM
- **Multi-Agent Systems**: Microsoft Autogen, Azure AI Foundry Agents
- **Responsible AI**: RAI evaluation metrics, safety assessments, compliance monitoring

### Azure AI Services
- **Azure OpenAI Service**: Latest model deployments and API integrations
- **Azure AI Foundry**: Agent development, evaluation, and deployment
- **Azure Machine Learning**: Model fine-tuning, deployment, and management
- **Azure AI Studio**: Managed endpoints and UI-based AI development
- **Azure AI Search**: RAG (Retrieval-Augmented Generation) implementations

### Development Approaches
- **SDK-First Development**: Python-based implementations using Azure AI SDKs
- **UI-Driven Solutions**: Azure AI Studio and Autogen Studio examples
- **Streamlit Applications**: Interactive web interfaces for AI models
- **Evaluation & Testing**: Comprehensive evaluation frameworks and simulation tools

## 📁 Repository Structure

### [AIFoundry/](./AIFoundry/) - Azure AI Foundry Examples
Advanced agent development and evaluation using Azure AI Foundry platform.

**Key Samples:**
- [`aifoundryraiagent.md`](./AIFoundry/aifoundryraiagent.md) - Responsible AI agent with manufacturing compliance
- [`agenteval.md`](./AIFoundry/agenteval.md) - Agent evaluation frameworks and metrics
- [`modelrouter.md`](./AIFoundry/modelrouter.md) - Intelligent model routing strategies
- [`redteam.md`](./AIFoundry/redteam.md) - Red team testing for AI safety

### [AOAI/](./AOAI/) - Azure OpenAI Service Examples
Direct integration examples with Azure OpenAI models for various enterprise scenarios.

**Key Samples:**
- [`o3mini.md`](./AOAI/o3mini.md) - Interactive Streamlit app with o3-mini model
- [`o1hospitalplan.md`](./AOAI/o1hospitalplan.md) - Healthcare capacity planning with o1 reasoning
- [`o1supplychain.md`](./AOAI/o1supplychain.md) - Supply chain optimization using o1 model
- [`o1mfgprodplanning.md`](./AOAI/o1mfgprodplanning.md) - Manufacturing production planning

### [autogen/](./autogen/) - Microsoft Autogen Multi-Agent Systems
Multi-agent orchestration for complex workflows and automated processes.

**Key Samples:**
- [`contenemail.md`](./autogen/contenemail.md) - Web content summarization with email distribution
- [`prdschedule.md`](./autogen/prdschedule.md) - Automated production scheduling system
- [`papersum.md`](./autogen/papersum.md) - Academic paper summarization workflows
- [`codeconv.md`](./autogen/codeconv.md) - Code to Autogen Studio configuration conversion

### [AML/](./AML/) - Azure Machine Learning Examples
Model deployment, fine-tuning, and management using Azure ML platform.

**Key Samples:**
- [`phi4reasning.md`](./AML/phi4reasning.md) - Phi-4 model reasoning capabilities
- [`deepseekllama8b.md`](./AML/deepseekllama8b.md) - DeepSeek and Llama model comparisons
- [`finetuneqlorallama3.2-1B.md`](./AML/finetuneqlorallama3.2-1B.md) - QLoRA fine-tuning techniques
- [`simulation.md`](./AML/simulation.md) - Data simulation for model testing and training

### [AgentsAI/](./AgentsAI/) - Specialized AI Agents
Domain-specific AI agents for various industry applications.

**Key Samples:**
- [`mfgagents.md`](./AgentsAI/mfgagents.md) - Manufacturing-focused AI agents
- [`agenticaigov.md`](./AgentsAI/agenticaigov.md) - AI governance and compliance agents
- [`deepseekR1modelcatalog.md`](./AgentsAI/deepseekR1modelcatalog.md) - DeepSeek R1 model deployment
- [`TechIdeationAgent.md`](./AgentsAI/TechIdeationAgent.md) - Technology ideation and innovation agents

### [AIStudio/](./AIStudio/) - Azure AI Studio Examples
UI-driven AI development and managed service examples.

**Key Samples:**
- [`phi4managedendpoint.md`](./AIStudio/phi4managedendpoint.md) - Managed endpoint deployment
- [`o1SupplyChain.md`](./AIStudio/o1SupplyChain.md) - Supply chain analysis using AI Studio

### [AIFunctions/](./AIFunctions/) - AI Function Tools
Reusable AI function implementations and tool integrations.

### [Concepts/](./Concepts/) - Conceptual Frameworks
High-level conceptual discussions and future-looking AI implementations.

**Key Samples:**
- [`mfgplantfloorai.md`](./Concepts/mfgplantfloorai.md) - Manufacturing plant floor AI concepts
- [`spacegpts.md`](./Concepts/spacegpts.md) - Space exploration AI applications

## 🛠️ Prerequisites

### Azure Services
- **Azure Subscription** with appropriate permissions
- **Azure OpenAI Service** access in supported regions (East US, Sweden Central)
- **Azure AI Foundry** workspace and project
- **Azure Machine Learning** workspace (for AML examples)
- **Azure AI Search** service (for RAG implementations)

### Development Environment
- **Python 3.11+** with virtual environment management
- **Visual Studio Code** or preferred IDE
- **Git** for version control
- **Azure CLI** for service management

### Key Python Packages
```bash
# Core AI packages
pip install azure-ai-projects azure-ai-evaluation azure-ai-inference
pip install openai azure-openai

# Autogen packages
pip install autogen-agentchat autogenstudio autogen-ext[azure]

# Web framework
pip install streamlit

# Monitoring and evaluation
pip install azure-monitor-opentelemetry
```

## 🚦 Getting Started

### 1. Environment Setup
```bash
# Clone the repository
git clone https://github.com/balakreshnan/Samples2025.git
cd Samples2025

# Set up Python environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies (varies by sample)
pip install -r requirements.txt  # If available for specific samples
```

### 2. Configure Azure Services
```bash
# Set environment variables for Azure services
export AZURE_OPENAI_ENDPOINT="https://your-aoai-resource.openai.azure.com/"
export AZURE_OPENAI_API_KEY="your-api-key"
export AZURE_AI_FOUNDRY_PROJECT_ID="your-project-id"
```

### 3. Choose Your Learning Path

**For Beginners:**
1. Start with [AOAI/o3mini.md](./AOAI/o3mini.md) for basic Azure OpenAI integration
2. Try [autogen/contenemail.md](./autogen/contenemail.md) for multi-agent introduction
3. Explore [AIFoundry/agenteval.md](./AIFoundry/agenteval.md) for evaluation concepts

**For Enterprise Developers:**
1. Review [AIFoundry/aifoundryraiagent.md](./AIFoundry/aifoundryraiagent.md) for responsible AI
2. Implement [AgentsAI/mfgagents.md](./AgentsAI/mfgagents.md) for industry applications
3. Study [AML/simulation.md](./AML/simulation.md) for evaluation frameworks

**For Advanced Practitioners:**
1. Explore [Concepts/mfgplantfloorai.md](./Concepts/mfgplantfloorai.md) for architectural patterns
2. Implement [autogen/prdschedule.md](./autogen/prdschedule.md) for complex orchestration
3. Review evaluation and governance examples in AIFoundry

## 📈 Use Cases & Business Value

### Manufacturing & Industrial
- **Production Planning**: Automated scheduling and capacity optimization
- **Compliance Management**: OSHA and NIST standards compliance verification
- **Quality Control**: AI-driven inspection and anomaly detection
- **Supply Chain**: End-to-end optimization and risk management

### Healthcare & Life Sciences
- **Capacity Planning**: Hospital resource allocation and patient flow optimization
- **Clinical Decision Support**: Evidence-based treatment recommendations
- **Research Acceleration**: Automated literature review and analysis

### Technology & Innovation
- **Code Generation**: Automated code creation and conversion
- **Research Synthesis**: Academic paper summarization and insight extraction
- **Ideation Support**: Technology innovation and concept development

## 🔧 Sample Categories

### By Complexity Level
- **🟢 Beginner**: Basic API integrations and simple Streamlit apps
- **🟡 Intermediate**: Multi-agent systems and evaluation frameworks
- **🔴 Advanced**: Complex orchestration and enterprise governance

### By Azure Service
- **Azure OpenAI**: Direct model integration and API usage
- **Azure AI Foundry**: Agent development and evaluation platforms
- **Azure ML**: Model management and deployment workflows
- **Azure AI Studio**: UI-driven development and managed services

### By Industry Vertical
- **Manufacturing**: Production, compliance, and operations optimization
- **Healthcare**: Planning, decision support, and research assistance
- **Technology**: Development acceleration and innovation support
- **General Business**: Process automation and decision enhancement

## 🤝 Contributing

This repository welcomes contributions! Whether you're:
- Adding new sample implementations
- Improving existing documentation
- Reporting issues or suggesting enhancements
- Sharing best practices and lessons learned

Please ensure your contributions:
- Include clear documentation and prerequisites
- Follow existing naming and structure conventions
- Provide practical, working examples
- Consider responsible AI principles

## 📚 Additional Resources

### Documentation
- [Azure OpenAI Service Documentation](https://docs.microsoft.com/azure/cognitive-services/openai/)
- [Azure AI Foundry Documentation](https://docs.microsoft.com/azure/ai-foundry/)
- [Microsoft Autogen Documentation](https://github.com/microsoft/autogen)

### Community
- [Azure AI Community](https://techcommunity.microsoft.com/azure/ai/)
- [Microsoft AI Blog](https://blogs.microsoft.com/ai/)
- [GitHub Discussions](../../discussions) - For questions and community support

### Learning Paths
- [Azure AI Fundamentals](https://docs.microsoft.com/learn/paths/get-started-with-artificial-intelligence-on-azure/)
- [Responsible AI Practices](https://docs.microsoft.com/azure/machine-learning/concept-responsible-ai)
- [Multi-Agent System Design](https://github.com/microsoft/autogen/tree/main/samples)

## ⚠️ Important Notes

- **Not Production Ready**: These samples are for learning and demonstration purposes
- **API Keys**: Never commit API keys or sensitive credentials to version control
- **Costs**: Azure AI services incur costs - monitor usage and set appropriate limits
- **Responsible AI**: Always implement appropriate safety measures and evaluation frameworks
- **Compliance**: Ensure adherence to your organization's AI governance policies

## 📄 License

This project may contain trademarks or logos for projects, products, or services. Use of Microsoft trademarks or logos must follow Microsoft's Trademark & Brand Guidelines. Use of such trademarks or logos in modified versions of this project must not cause confusion or imply Microsoft sponsorship.

---

**🎯 Ready to explore the future of AI?** Choose a sample that matches your interests and start building with the latest AI technologies today!
