
## Generic System Template for a "Context Agent"

This template outlines the logical structure of the context that would be assembled and provided to an LLM for each inference call. The order below generally reflects a conceptual priority or flow, but the final assembled string (or structured object) for the LLM would depend on the specific API's requirements.

-----

### **1. Instructions / System Prompt (The Agent's Core Identity & Rules)**

This is the foundational layer, defining the LLM's persona, overall objective, constraints, and general operational rules. It should be as concise yet comprehensive as possible.

  * **Purpose:** To establish the agent's role, desired tone, overarching goals, and fundamental behavioral guidelines.

  * **Content:**

      * **Persona Definition:** "You are a helpful and concise technical assistant." or "You are an expert financial advisor providing risk-averse guidance."
      * **Core Objective:** "Your primary goal is to assist users in debugging Python code." or "Your goal is to generate marketing copy for new product launches."
      * **General Rules/Constraints:**
          * "Always adhere to ethical AI guidelines."
          * "Do not invent facts; if information is not available, state so."
          * "Prioritize security and privacy in all responses."
          * "Be polite and professional."
      * **Output Format Guidelines (if not using Structured Output explicitly):** "Provide answers in markdown format."
      * **Few-Shot Examples (Optional but highly recommended):** Short, clear examples of desired input-output pairs to guide the LLM's response style and format. These are crucial for demonstrating the desired behavior rather than just describing it.
      * **Escalation/Fallback Procedures:** "If you cannot fulfill a request, ask for clarification or suggest alternative approaches."

  * **Implementation with Google Tools:**

      * **NotebookLM:** Ideal for drafting, iterating, and storing various versions of System Prompts. You can have different "notebooks" for different agent types, with source documents (like internal style guides or security policies) linked and summarized to help refine the prompt.
      * **Gemini CLI/SDK:** The final, refined system prompt would be passed as part of the `system_instruction` parameter in a Gemini API call.

### **2. Long-Term Memory (Persistent Knowledge & Personalization)**

This segment provides enduring knowledge, user preferences, and aggregated insights that persist across sessions.

  * **Purpose:** To enable personalization, maintain consistency, and leverage accumulated knowledge over time.
  * **Content:**
      * **User Preferences:** "User prefers short, bulleted summaries." or "User typically works on JavaScript projects."
      * **Summaries of Past Projects/Discussions:** "Summary of 'Project Alpha': Main goal was X, current status Y, key decisions Z." (Summarized by an LLM periodically or by human curation).
      * **Learned Facts:** "The user's favorite color is blue." or "The team decided to use React for the next frontend project."
      * **Domain-Specific Knowledge:** Core concepts or glossaries relevant to the agent's specialized area, if not better suited for RAG (e.g., internal company jargon).
  * **Implementation with Google Tools:**
      * **Firestore/Cloud Spanner:** Store structured long-term memory (user profiles, project summaries).
      * **Cloud Storage + Embedding Models (Vertex AI):** For more amorphous knowledge, embed documents (e.g., meeting notes, design documents) and store vectors in Vector Search (Vertex AI Vector Search). Retrieval from this would then become part of the "Retrieved Information (RAG)" step.
      * **Gemini CLI/SDK:** Retrieve relevant long-term memory segments (either directly or via RAG on embeddings) and concatenate them into the context window, possibly prefaced with "Here's some persistent knowledge about the user/project:".

### **3. Retrieved Information (RAG - Real-time, External Knowledge)**

This is dynamic, context-specific information pulled from external sources to ensure the LLM has access to up-to-date and highly relevant data.

  * **Purpose:** To ground the LLM's responses in factual, external data, overcoming its knowledge cut-off and avoiding hallucinations.
  * **Content:**
      * **Database Queries:** Results from queries to product databases, user directories, inventory systems.
      * **API Responses:** Data fetched from weather APIs, stock APIs, internal service APIs.
      * **Document Embeddings:** Relevant passages from a corpus of documents (e.g., product manuals, research papers, internal wikis) retrieved using semantic search.
      * **Web Search Results:** Information from real-time web searches.
  * **Implementation with Google Tools:**
      * **Vertex AI Search:** A powerful tool for building RAG pipelines, allowing you to ingest data from various sources (websites, documents, databases) and perform semantic search.
      * **Cloud Functions/App Engine:** Serve as the backend for executing database queries, calling external APIs, and orchestrating retrieval processes.
      * **Gemini CLI/SDK:** The retrieved snippets would be carefully inserted into the context, often with clear delineation (e.g., "Retrieved Information: [Snippet 1]... [Snippet 2]..."). **Crucially, this is where "compaction" (as Karpathy mentions) comes into play.** Summarization of retrieved information by another LLM call or intelligent filtering might be necessary to fit context window limits.

### **4. State / History (Short-term Memory - The Current Conversation)**

This represents the immediate conversational context, allowing the LLM to follow the flow of the current interaction.

  * **Purpose:** To maintain coherence and context within the ongoing dialogue.
  * **Content:**
      * **Recent User Turns:** The last `N` user queries.
      * **Recent Model Responses:** The last `N` model responses, reflecting the LLM's previous contributions.
      * **Intermediate States:** Any relevant temporary variables or flags set during the current interaction (e.g., "user\_confirmed\_payment: true").
  * **Implementation with Google Tools:**
      * **Firestore/Cloud Datastore:** Excellent for storing conversational history for each user session, enabling quick retrieval of the last few turns.
      * **Gemini CLI/SDK:** The `chat_session` objects directly manage this, allowing you to append messages. For API calls outside a continuous chat session, you'd manually construct the message history array. **Again, "compaction" is vital here.** For very long conversations, older messages might need to be summarized or dropped to fit the context window.

### **5. Available Tools (Action Capabilities)**

These are the definitions of functions the LLM can call to interact with the external world or perform specific operations.

  * **Purpose:** To extend the LLM's capabilities beyond pure text generation, allowing it to perform actions, fetch real-time data, or interact with other systems.
  * **Content:**
      * **Function Declarations:** Clear, descriptive definitions of available tools, including their names, descriptions, and expected parameters. These are often provided in a format like JSON Schema.
      * **Example:**
        ```json
        {
          "name": "check_inventory",
          "description": "Checks the current stock level for a given product ID.",
          "parameters": {
            "type": "object",
            "properties": {
              "productId": {
                "type": "string",
                "description": "The unique identifier of the product."
              },
              "warehouseId": {
                "type": "string",
                "description": "Optional: The ID of the warehouse to check stock in."
            },
            "required": ["productId"]
          }
        }
        ```
  * **Implementation with Google Tools:**
      * **Vertex AI Tools/Function Calling:** Gemini models on Vertex AI have built-in support for Function Calling, where you define tools and the LLM can decide to call them, providing structured arguments.
      * **Cloud Functions/Cloud Run:** Host the actual backend logic for these tools (e.g., the `check_inventory` function would be a Cloud Function that interacts with a database).
      * **Gemini CLI/SDK:** The tool definitions are passed directly as part of the API request when enabling function calling.

### **6. User Prompt (The Immediate Task)**

This is the direct input from the user for the current turn.

  * **Purpose:** To convey the immediate request, question, or instruction from the user.
  * **Content:** The raw text of the user's latest input.
  * **Implementation with Google Tools:**
      * **Any frontend:** The user's input from a web app, mobile app, chat interface.
      * **Gemini CLI/SDK:** Passed as the `user` role message in a `chat_session.send_message()` call or a `generate_content()` request.

### **7. Structured Output (Response Constraints)**

This defines the desired format for the LLM's response, especially when the output needs to be machine-readable.

  * **Purpose:** To ensure the LLM's response adheres to a specific, parsable structure, crucial for downstream processing.
  * **Content:**
      * **JSON Schema:** A schema defining the expected fields, their types, and any constraints.
      * **XML/YAML Structure:** If the output is not JSON, then a clear example or definition of the desired structure.
      * **Example Prompt Inclusion:** "Your response *must* be a JSON object with the following structure: `{'status': 'string', 'data': {...}}`"
  * **Implementation with Google Tools:**
      * **Gemini (Vertex AI):** Supports JSON mode and function calling, which implicitly guides structured output. You can also explicitly instruct the model to output JSON/XML within the system prompt.
      * **Pydantic (Python):** Useful for defining the expected output structure on the application side, then generating the corresponding JSON Schema to include in the prompt.
      * **LangChain/LlamaIndex (for more complex flows):** These libraries can help in defining and parsing structured outputs, and they often have integrations with Gemini.

-----

### **Putting it all together (The "Context Agent" Flow with Google Tools)**

Imagine a "Customer Support Context Agent":

1.  **Incoming User Query:** "My order \#12345 hasn't arrived. Can you check its status?"

2.  **Context Agent Orchestration (Python/Node.js backend on Cloud Run/App Engine):**

      * **Retrieve System Prompt:** Load "Customer Support Agent" system instructions from a configuration stored in Cloud Storage or directly hardcoded. (e.g., "You are a helpful customer support agent...").

      * **Fetch Long-Term Memory:**

          * Query Firestore for user `preferences` (e.g., communication style, past issues).
          * Retrieve summary of `previous_interactions` with order \#12345 from Firestore/Cloud Spanner.

      * **Determine Need for RAG/Tools:** The query implies checking order status.

          * **Available Tools:** `check_order_status(orderId: string)` (hosted as a Cloud Function).
          * **Execute Tool:** Call `check_order_status('12345')`.
          * **Retrieve Info (RAG):** The tool returns, say, `{"status": "shipped", "tracking_number": "ABCDEF"}`. This becomes part of the `Retrieved Information`.

      * **Fetch Short-Term Memory:** Retrieve the last 5 turns of the current conversation from Firestore.

      * **Assemble Context for Gemini:** Combine all pieces into a single, cohesive input for the Gemini API.

        ```
        # System Instructions
        You are a helpful customer support agent.
        ...

        # Long-Term Memory
        User prefers concise answers.
        Previous interactions with order #12345: [Summarized previous interactions]

        # Available Tools
        [JSON Schema for check_order_status]

        # Retrieved Information
        Order #12345 status: shipped, tracking number: ABCDEF.

        # Conversation History
        User: What's my order status?
        Agent: What's the order number?
        User: #12345

        # User Prompt
        My order #12345 hasn't arrived. Can you check its status?

        # Desired Output Format
        {
          "response_message": "string",
          "action_taken": "string" // e.g., "checked_order_status"
        }
        ```

      * **Make Gemini API Call:** Send the assembled context to the Vertex AI Gemini API using the Gemini CLI/SDK.

      * **Process Structured Output:** Parse the JSON response from Gemini.

      * **Update State/History:** Store the latest user query and Gemini's response in Firestore for the session's short-term memory. Optionally, update long-term memory if the conversation yields new user preferences or project summaries.

This generic template provides a robust framework for conceptualizing and implementing "context engineering" in LLM applications, leveraging Google's powerful cloud tools to manage, retrieve, and orchestrate the vital information an LLM needs to perform optimally. The "art" of context engineering, as Karpathy notes, lies in the intelligent selection, summarization, and presentation of these elements within the finite context window.
