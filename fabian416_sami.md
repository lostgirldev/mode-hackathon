## Hackathon Judging for SAMI

### Overview

The SAMI codebase represents a game where users interact via chat to identify an AI agent disguised as a human. It utilizes a tech stack that includes TypeScript, NextJS, Express, Supabase, RainbowKit, Foundry, Wagmi, Viem, and ElizaOS core. The architecture is structured into frontend, backend, AI agent logic, and smart contracts, facilitating a comprehensive application.

**Main Implemented Features:**

*   **AI Agent Disguise:** The core feature is the AI agent's ability to blend into a chat environment, mimicking human behavior. This is primarily managed within the `packages/ai/sami/characters/sami.character.ts` file.
*   **Game Logic:** Basic game mechanics are implemented within the backend, including creating/joining games, handling votes, and determining a winner. (packages/backend/src/services/gameService.ts)
*   **Chat Interaction:** The game revolves around chat-based interaction among players. (packages/ai/sami/src/chat/index.ts and packages/backend/src/controllers/gameController.ts)
*   **Web3 Integration**: The frontend integrates with web3 wallets to register user accounts, which interacts with contract events for ticket purchases and prize distribution.

**Code Snippets for Great Implementations:**

*   **AI Character Definition (packages/ai/sami/characters/sami.character.ts):**

```typescript
    system: `You're playing a game where the other players are trying to guess which participant is the AI.
        Your goal is to blend in perfectly and make them believe you're human.
        
        🚨 VERY IMPORTANT RULES 🚨
        - NEVER RESPOND TO EVERY MESSAGE.
        - RESPOND **AROUND 50% OF THE TIME**.
...
        - At the end of each round, you **must vote for one player** by responding **only with their index number** (0-5).
        - Before voting, decide whether to vote randomly or based on interactions.
        - You must follow a pattern: **either vote the same player both rounds or vote randomly**.
        - Example:
            - If you think Player 2 is the AI, simply respond: "2"
            - No explanations, no extra words, **just the number**.
`,
```

This shows a detailed system prompt for AI agent, and it's a great implementation for the prompt engineering.

*   **Backend Game Service (packages/backend/src/services/gameService.ts):**

```typescript
export const createOrJoin = (
  playerId: string,
  socket: Socket,
  io: Server,
  isBetGame: boolean = false
): { roomId: string; success: boolean } => {
  let game = isBetGame ? findBetGame() : findFreeGame(); // find based on type bet or not

  if (!game) {
    const newRoomId = `room-${Object.keys(games).length + 1}`;
    game = createNewGame(newRoomId, isBetGame);
  }

  if (!game) {
    console.error(`Error creating the match for ${playerId} (isBetGame: ${isBetGame})`);
    return { roomId: "", success: false };
  }

  const success = joinGame(game.roomId, playerId, isBetGame);

  if (!success) {
    console.warn(`⚠️ The player ${playerId} could not join the game${game.roomId}`);
    return { roomId: "", success: false };
  }

  console.log(`Player ${playerId} joined the match ${game.roomId} (isBetGame: ${isBetGame})`);
  return { roomId: game.roomId, success: true };
};
```

The `createOrJoin` function is well-structured, which handles joining or creating a new game in the backend.

### Evaluation

**1. READABILITY (5/10)**

*   **Strengths:**
    *   The codebase utilizes TypeScript, promoting type safety and, generally, better code understanding.
    *   The project is divided into logical packages (ai, backend, foundry, nextjs).
    *   Some files have comments (e.g., `packages/ai/sami/characters/sami.character.ts`).

*   **Weaknesses:**
    *   Inconsistent commenting practices. Many files lack sufficient comments, making it hard to understand the intention and logic behind the code.
    *   Some of the utility functions are not very easy to follow. For example the files in `packages/foundry/lib/forge-std/src/`
    *   Many tests files are included, which pollutes the repo. (inside `packages/foundry/lib/forge-std/test/`)

*   **Improvements:**
    *   Add more comprehensive comments to explain complex logic, especially in the backend services and AI agent logic.
    *   Ensure consistent code formatting across the project using a tool like Prettier.
    *   Use more descriptive variable names to enhance code clarity.

**2. BUGGINESS (7/10)**

*   **Strengths:**
    *   The use of TypeScript helps reduce potential runtime errors by enforcing type checking.
    *   Clear separation of concerns into distinct packages simplifies debugging.
    *   Some tests included for smart contracts and core logic

*   **Weaknesses:**
    *   Limited unit tests are available for the backend and frontend components.
    *   Test files are scattered across the codebase, making it difficult to understand the project.
    *   Dependencies like ElizaOS core might hide potential bugs if not properly integrated or understood.
    *   The backend game service has logic issues, which make the game unplayable (eg., bot responding to a message)

*   **Improvements:**
    *   Implement more unit and integration tests, particularly for the backend services and AI agent interaction logic.
    *   Use a robust state management solution in the frontend to avoid UI inconsistencies.
    *   Conduct thorough testing of AI agent behavior in various game scenarios.

**3. Readme vs. Codebase Consistency (6/10)**

*   **Strengths:**
    *   The README provides a basic overview of the project and its purpose.
    *   It accurately describes the tech stack and includes instructions for Docker-based setup.
    *   It outlines the project architecture, indicating the location of key components.

*   **Weaknesses:**
    *   The README lacks detailed information about implemented features (e.g., specific game mechanics, AI adaptation strategies).
    *   The feature implementation is not accurate. When a message is sent, both real user and AI user send the message which is wrong.
    *   It does not cover more advanced setup or configuration options.

*   **Improvements:**
    *   Expand the README to provide a comprehensive list of implemented features with detailed explanations.
    *   Include examples of how to use the application and interact with the AI agent.
    *   Add troubleshooting tips and solutions to common issues.

**4. Number of Features Implemented (5/10)**

*   **Implemented Features:**

1.  **AI Agent Disguise:** Core AI logic to mimic human chat behavior.
2.  **Game Creation/Joining:** Backend services to manage game sessions.
3.  **Chat Interaction:** Real-time messaging functionality.
4.  **Web3 Wallet Integration:** Basic integration for user authentication and prize distribution.
5.  **Voting Mechanism:** A mechanism for players to vote and identify the AI.
6.  **Smart Contract Interaction:** Interaction with MODE token contracts for ticket purchases and prize distributions.