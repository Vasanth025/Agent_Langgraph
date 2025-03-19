Below is a **README** specifically for the AI Agent (`aiAgent.js`) in your travel assistant project, focusing solely on its functionality without references to the database (Pinecone), historical data, or data feeding processes. The README describes the agent's purpose, setup, usage, and code structure, tailored to its role in generating travel itineraries using Mistral AI and LangChain.

---

# Travel Itinerary AI Agent

## Overview

The Travel Itinerary AI Agent (`aiAgent.js`) is a Node.js module designed to generate personalized travel itineraries based on user queries (e.g., "3-day trip to Ooty"). It leverages the Mistral AI language model for natural language processing (NLP) and LangChain for prompt management. The agent operates within a larger travel assistant system, receiving user inputs and producing structured itineraries as output.

This module focuses solely on itinerary generation, relying on the Mistral AI model to interpret user queries and create travel plans. It does not handle data storage, historical data retrieval, or external data feeding.

## Features

- **Itinerary Generation:** Creates detailed travel itineraries based on user queries.
- **Natural Language Processing:** Uses Mistral AI to understand user inputs and generate human-readable responses.
- **Prompt Engineering:** Employs LangChain to structure prompts for consistent output formatting.
- **Modular Design:** Integrates seamlessly with a LangGraph workflow for state management and orchestration.

## Prerequisites

- **Node.js:** Version 18.x or higher.
- **npm:** For package management.
- **Mistral AI API Key:** Required for accessing the Mistral AI model.
- **Environment Variables:** A `.env` file with the following:
  ```
  MISTRAL_API_KEY=your_mistral_api_key
  MODEL_NAME=mistral-tiny
  TEMPERATURE=0.7
  NODE_ENV=development
  ```

## Installation

1. **Clone the Repository (or navigate to the project directory):**
   ```bash
   git clone <repository-url>
   cd <project-directory>
   ```

2. **Install Dependencies:**
   The AI Agent depends on the following packages, which should be installed in the project root:
   ```bash
   npm install @langchain/mistralai dotenv
   ```
   - `@langchain/mistralai`: For interacting with the Mistral AI model.
   - `dotenv`: For loading environment variables.

3. **Set Up Environment Variables:**
   Create a `.env` file in the project root with the required variables (see Prerequisites).

4. **Verify Model Setup:**
   Ensure the `utils/model.js` file (below) is present to initialize the Mistral AI model.

## Code Structure

### `utils/model.js`
This utility file initializes the Mistral AI model for use by the AI Agent.

```javascript
import { ChatMistralAI } from "@langchain/mistralai";
import dotenv from "dotenv";

// Load environment variables
dotenv.config();

// Validate API key
const MISTRAL_API_KEY = process.env.MISTRAL_API_KEY;
if (!MISTRAL_API_KEY) {
  throw new Error("MISTRAL_API_KEY is missing in .env file. Please provide a valid API key.");
}

// Define model name
const modelName = process.env.MODEL_NAME || "mistral-tiny";

// Initialize the Mistral AI model
export const model = new ChatMistralAI({
  apiKey: MISTRAL_API_KEY,
  modelName: modelName,
  temperature: parseFloat(process.env.TEMPERATURE) || 0.7,
  cache: process.env.NODE_ENV !== "development",
  maxRetries: 3,
  timeout: 10000,
});

// Log model initialization in development
if (process.env.NODE_ENV === "development") {
  console.log(`Mistral AI model initialized: ${modelName}`);
}
```

### `aiAgent.js`
The core AI Agent module that generates travel itineraries.

```javascript
import { model } from "./utils/model.js";

export async function itineraryNode(state) {
  if (!state.query) throw new Error("Query is required.");
  const prompt = `Create a travel itinerary with places to visit based on this plan: ${state.query}`;
  const { content } = await model.invoke(prompt);
  return { itinerary: content };
}

export default itineraryNode;
```

- **Functionality:** The `itineraryNode` function takes a `state` object with a `query` (e.g., "3-day trip to Ooty"), constructs a prompt, and uses Mistral AI to generate an itinerary.
- **Input:** A `state` object with a `query` field (string).
- **Output:** An updated `state` object with an `itinerary` field (string).

## Usage

The AI Agent is designed to be used within a LangGraph workflow (`StateGraph`). Below is an example of how to integrate it into a larger system.

### Example Integration

1. **Define the State Schema (state.js):**
   ```javascript
   import { Annotation } from "@langchain/langgraph";

   export const TravelState = {
     query: Annotation({
       reducer: (x, y) => y ?? x,
       default: () => "",
     }),
     itinerary: Annotation({
       reducer: (x, y) => y ?? x,
       default: () => null,
     }),
   };
   ```

2. **Set Up the Workflow (travelGraph.js):**
   ```javascript
   import { StateGraph } from "@langchain/langgraph";
   import { itineraryNode } from "./aiAgent.js";
   import { TravelState } from "./state.js";

   const graph = new StateGraph(TravelState)
     .addNode("itineraryNode", itineraryNode)
     .setEntryPoint("itineraryNode");

   export const travelWorkflow = graph.compile();
   ```

3. **Invoke the Workflow (index.js):**
   ```javascript
   import express from "express";
   import { travelWorkflow } from "./travelGraph.js";

   const app = express();
   app.use(express.json());

   app.post("/ask", async (req, res) => {
     const { query } = req.body;
     if (!query || typeof query !== "string") {
       return res.status(400).json({ success: false, error: "Query must be a non-empty string." });
     }

     try {
       const initialState = { query, itinerary: null };
       const result = await travelWorkflow.invoke(initialState);
       res.json({ success: true, data: { itinerary: result.itinerary } });
     } catch (error) {
       res.status(500).json({ success: false, error: `Error: ${error.message}` });
     }
   });

   const PORT = process.env.PORT || 5000;
   app.listen(PORT, () => console.log(`🚀 Travel Assistant running on port ${PORT}`));
   ```

4. **Test the Agent:**
   Send a POST request to `http://localhost:5000/ask`:
   ```json
   {
     "query": "3-day trip to Ooty"
   }
   ```
   **Expected Response:**
   ```json
   {
     "success": true,
     "data": {
       "itinerary": "- Day 1: Morning - Visit Ooty Lake...\n- Day 2: Explore Doddabetta Peak..."
     }
   }
   ```

## Development Notes

- **Environment Variables:** Ensure `MISTRAL_API_KEY` is set in `.env`. The `MODEL_NAME` can be adjusted (e.g., `mistral-tiny`, `mistral-medium`) based on your needs.
- **Error Handling:** The agent throws an error if `state.query` is missing, ensuring robust input validation.
- **Prompt Design:** The prompt in `itineraryNode` is simple but can be enhanced for more detailed outputs (e.g., specifying itinerary format or including additional constraints).

## Limitations

- **Dependency on Mistral AI:** Requires a stable internet connection and a valid API key.
- **No Data Storage:** This version of the agent does not interact with databases or store historical data.
- **Basic Prompting:** The prompt is straightforward; advanced use cases may require more complex prompt engineering.

## Future Improvements

- **Enhanced Prompting:** Add more detailed instructions to the prompt (e.g., "Include specific times, avoid crowded places").
- **Multi-Model Support:** Allow switching between different language models (e.g., adding support for OpenAI models).
- **Input Validation:** Expand validation to handle more query formats or constraints.

## License

This project is licensed under the MIT License. See the `LICENSE` file for details.

---

This README focuses solely on the AI Agent (`aiAgent.js`), its integration with Mistral AI, and its role in generating itineraries, excluding database interactions, historical data, and data feeding processes. Let me know if you need adjustments!
