### Overview

The codebase implements an AI assistant named MAFiA that interacts with the Mode testnet using OpenAI's models. It provides a conversational interface for blockchain operations, allowing users to check balances, send transactions, and deploy smart contracts through natural language instructions. The system uses Viem for blockchain interactions and OpenAI's API for the AI assistant functionality.

**Main Implemented Features:**

1.  **Conversational Interface:** Uses OpenAI's API to create an assistant and thread for conversation. The `src/index.ts` sets up a readline interface for user input, sends messages to the thread, and displays the assistant's responses.
2.  **Wallet Balance Check:** Implemented in `tools/getBalance.ts`. The `getBalanceTool` uses `createViemPublicClient` to connect to the Mode testnet and fetches the balance of a specified wallet address.
    ```typescript
    // tools/getBalance.ts
    export const getBalanceTool: ToolConfig<GetBalanceArgs> = {
      definition: { ... },
      handler: async ({ wallet }) => {
        return await getBalance(wallet);
      },
    };
    ```
3.  **Wallet Address Retrieval:** Implemented in `tools/getWalletAddress.ts`. The `getWalletAddressTool` uses `createViemWalletClient` to retrieve the connected wallet address.
    ```typescript
    // tools/getWalletAddress.ts
    export const getWalletAddressTool: ToolConfig<GetWalletAddressArgs> = {
      definition: { ... },
      handler: async () => {
        return await getWalletAddress();
      },
    };
    ```
4.  **Transaction Sending:** Implemented in `tools/sendTransction.ts`. The `sendTransactionTool` uses `createViemWalletClient` to send transactions with customizable parameters such as recipient address, value, data, gas price, and more.
    ```typescript
    // tools/sendTransction.ts
    export const sendTransactionTool: ToolConfig<SendTransactionArgs> = {
      definition: { ... },
      handler: async (args) => {
        const result = await sendTransaction(args);
        if (!result.success || !result.hash) throw new Error(result.message);
        return result.hash;
      },
    };
    ```
5.  **ERC20 Token Deployment:** Implemented in `tools/deployERC20.ts`. The `deployErc20Tool` uses `createViemWalletClient` to deploy ERC20 tokens with customizable name, symbol, and initial supply.
    ```typescript
    // tools/deployERC20.ts
    export const deployErc20Tool: ToolConfig = {
      definition: { ... },
      handler: async (args: {
        name: string;
        symbol: string;
        initialSupply?: string;
      }) => {
        // ...
        const hash = await walletClient.deployContract({
          account: walletClient.account,
          abi: ERC20_ABI,
          bytecode: ERC20_BYTECODE,
          args: [args.name, args.symbol, baseNumber],
        });
        // ...
      },
    };
    ```
6.  **OpenAI Assistant Creation**: The `src/openai/createAssistant.ts` file contains the logic to create an OpenAI assistant with a specific model ("gpt-4o-mini"), name ("Al Capone"), instructions (defined in `src/constants/prompt.ts`), and tools (defined in `tools/allTools.ts`).
    ```typescript
    // src/openai/createAssistant.ts
    export async function createAssistant(client: OpenAI): Promise<Assistant> {
      return await client.beta.assistants.create({
        model: "gpt-4o-mini",
        name: "Al Capone",
        instructions: assistantPrompt,
        tools: Object.values(tools).map((tool) => tool.definition),
      });
    }
    ```
7. **Thread and Run Management**: `src/openai/createThread.ts` and `src/openai/createRun.ts` handle the creation of OpenAI threads and runs, respectively. These are crucial for managing the conversational flow and executing tasks with the assistant.
    ```typescript
    // src/openai/createThread.ts
    export async function createThread(
      client: OpenAI,
      message?: string
    ): Promise<Thread> {
      const thread = await client.beta.threads.create();
      if (message) {
        await client.beta.threads.messages.create(thread.id, {
          role: "user",
          content: message,
        });
      }
      return thread;
    }

    // src/openai/createRun.ts
    export async function createRun(
      client: OpenAI,
      thread: Thread,
      assistantId: string
    ): Promise<Run> {
      let run = await client.beta.threads.runs.create(thread.id, {
        assistant_id: assistantId,
      });

      while (run.status === "in_progress" || run.status === "queued") {
        await new Promise((resolve) => setTimeout(resolve, 1000)); // Wait 1 second
        run = await client.beta.threads.runs.retrieve(thread.id, run.id);
      }

      return run;
    }
    ```
8. **Tool Handling and Execution**: `src/openai/handleRunCalls.ts` is responsible for handling tool calls made by the OpenAI assistant. It parses the arguments, executes the corresponding tool, and submits the results back to the OpenAI run.
    ```typescript
    // src/openai/handleRunCalls.ts
    export async function handleRunToolCalls(
      run: Run,
      client: OpenAI,
      thread: Thread
    ): Promise<Run> {
      const toolCalls = run.required_action?.submit_tool_outputs?.tool_calls;

      if (!toolCalls) return run;

      const toolOutputs = await Promise.all(
        toolCalls.map(async (tool) => {
          const toolConfig = tools[tool.function.name];
          if (!toolConfig) {
            console.error(`Tool ${tool.function.name} not found`);
            return null;
          }

          console.log(`💾 Executing: ${tool.function.name}`);

          try {
            const args = JSON.parse(tool.function.arguments);
            const output = await toolConfig.handler(args);
            return {
              tool_call_id: tool.id,
              output: String(output),
            };
          } catch (error) {
            const errorMessage =
              error instanceof Error ? error.message : String(error);
            return {
              tool_call_id: tool.id,
              output: `Error: ${errorMessage}`,
            };
          }
        })
      );

      const validOutputs = toolOutputs.filter(
        Boolean
      ) as OpenAI.Beta.Threads.Runs.RunSubmitToolOutputsParams.ToolOutput[];
      if (validOutputs.length === 0) return run;

      return client.beta.threads.runs.submitToolOutputsAndPoll(thread.id, run.id, {
        tool_outputs: validOutputs,
      });
    }
    ```
9. **Run Execution and Result Retrieval**: The `src/openai/performRuns.ts` file contains the logic to perform an OpenAI run, handle tool calls, and retrieve the assistant's message.
    ```typescript
    // src/openai/performRuns.ts
    export async function performRun(run: Run, client: OpenAI, thread: Thread) {
      console.log(`🚀 Performing run ${run.id}`);

      while (run.status === "requires_action") {
        run = await handleRunToolCalls(run, client, thread);
      }

      if (run.status === "failed") {
        const errorMessage = `I encountered an error: ${
          run.last_error?.message || "Unknown error"
        }`;
        console.error("Run failed:", run.last_error);
        await client.beta.threads.messages.create(thread.id, {
          role: "assistant",
          content: errorMessage,
        });
        return {
          type: "text",
          text: {
            value: errorMessage,
            annotations: [],
          },
        };
      }

      const messages = await client.beta.threads.messages.list(thread.id);
      const assistantMessage = messages.data.find(
        (message) => message.role === "assistant"
      );

      console.log(`🚀 Assistant message: ${assistantMessage?.content[0]}`);

      return (
        assistantMessage?.content[0] || {
          type: "text",
          text: { value: "No response from assistant", annotations: [] },
        }
      );
    }
    ```
### Criteria Evaluation

*   **READABILITY:** 8/10

    *   **Reasoning:** The code is generally well-structured and uses meaningful variable names, which enhances readability. The modular design, with separate files for different functionalities (e.g., OpenAI interaction, Viem client creation, tool definitions), also contributes positively. However, there's room for improvement with more detailed comments, especially within the tool handler functions.
    *   **Strengths:**
        *   Clear file and folder structure.
        *   Meaningful variable and function names.
        *   Modular design.
    *   **Weaknesses:**
        *   Some areas lack sufficient comments, making it harder to quickly understand the purpose of certain code blocks.
        *   Inconsistencies in commenting style (some files are well-commented, others are not).
    *   **Improvements:**
        *   Add more comments, especially in complex logic sections, to explain the purpose and functionality.
        *   Enforce a consistent commenting style across the codebase.

*   **BUGGINESS:** 9/10

    *   **Reasoning:** The code appears to implement the described functionality correctly. The use of Viem and OpenAI's API suggests a reasonable degree of stability. The code has well-defined functions and clear control flow, reducing the likelihood of bugs.
    *   **Strengths:**
        *   Clear separation of concerns.
        *   Usage of established libraries (Viem, OpenAI).
        *   Well-defined function and tool interfaces.
    *   **Weaknesses:**
        *   No explicit error handling in main functionalities.
    *   **Improvements:**
        *   Address potential edge cases by handling them gracefully.
        *   Implement comprehensive testing to ensure all functions work as expected under different conditions.

*   **Readme vs. Codebase Consistency:** 9/10

    *   **Reasoning:** The README accurately reflects the implemented features. All claimed features (wallet operations, transaction management, smart contract interaction, conversational interface) are implemented in the codebase and function as described. The sequence diagram provides a good overview of the application flow.
    *   **Strengths:**
        *   All features described in the README are implemented.
        *   The codebase structure aligns with the description in the README.
        *   The sequence diagram accurately represents the application flow.
    *   **Weaknesses:**
        *   The README could benefit from more detailed usage examples.
    *   **Improvements:**
        *   Add specific examples of how to use the tool, including example conversations and expected outputs.

*   **Number of features implemented:** 8/10

    *   **Reasoning:** The codebase implements a solid set of core features, including conversational interaction, wallet operations (balance check, address retrieval), transaction sending, and ERC20 token deployment.
    *   **Strengths:**
        *   Implements the core features described in the README.
        *   Provides a good foundation for further feature development.
    *   **Weaknesses:**
        *   Could benefit from additional features.
    *   **Improvements:**
        *   Implement additional features, such as:
            *   Interaction with existing contracts (beyond just deployment).
            *   More sophisticated transaction parameter handling.
            *   Support for other token standards (e.g., ERC721).
