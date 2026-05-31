# 🟣 Intercom | Minicom Round — Full-Stack Preparation Guide

> **Round:** 3A — Minicom (90 min: 45 solo + 45 pairing)
> **What it is:** Build a feature in a pre-existing messaging app (mini Intercom)
> **Tech Stack:** React + TypeScript (frontend) | Node.js/Express OR Ruby on Rails (backend) | PostgreSQL
> **Key Insight:** "Focus on shipping code rather than code quality"

---

## 📋 Table of Contents

1. [Round Format & Strategy](#round-format--strategy)
2. [Frontend Patterns You MUST Know](#frontend-patterns-you-must-know)
3. [Backend Patterns You MUST Know](#backend-patterns-you-must-know)
4. [Practice Project: Mini Chat App](#practice-project-mini-chat-app)
5. [Common Minicom Tasks & Solutions](#common-minicom-tasks--solutions)
6. [Pairing Session Strategy](#pairing-session-strategy)
7. [Time Management Blueprint](#time-management-blueprint)

---

## Round Format & Strategy

### What Candidates Report

> "You receive a repo with an existing chat application. The frontend has React components, the backend has REST endpoints. You need to add a feature."

> "45 minutes alone, then 45 minutes pairing with an interviewer."

> "You are asked to complete one small requirement and then work on whatever you want with it."

### The Evaluation Criteria

| What They Check | Weight | How to Show It |
|----------------|--------|----------------|
| **Shipping working code** | 🔴 HIGH | Get something functional FAST |
| **Navigating unfamiliar codebase** | 🔴 HIGH | Spend first 10 min exploring structure |
| **Communication (pairing)** | 🔴 HIGH | Think aloud, ask questions, propose ideas |
| **Full-stack capability** | 🟠 MED | Touch both frontend AND backend |
| **Code quality** | 🟡 LOW | Don't polish — ship |

### Strategy

```
Minutes 0–10:  READ (README, folder structure, run the app, click around)
Minutes 10–15: PLAN (identify files to touch, mental model of data flow)
Minutes 15–45: BUILD (implement feature, test by refreshing UI)
Minutes 45–90: PAIR (interviewer joins — communicate, extend the feature)
```

---

## Frontend Patterns You MUST Know

### 1. React Functional Component + API Call

```tsx
import React, { useState, useEffect } from 'react';

interface Message {
  id: number;
  content: string;
  sender_id: number;
  conversation_id: number;
  created_at: string;
}

interface Props {
  conversationId: number;
}

const MessageList: React.FC<Props> = ({ conversationId }) => {
  const [messages, setMessages] = useState<Message[]>([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchMessages();
  }, [conversationId]);

  const fetchMessages = async () => {
    try {
      const res = await fetch(`/api/conversations/${conversationId}/messages`);
      const data = await res.json();
      setMessages(data);
    } catch (err) {
      console.error('Failed to fetch messages:', err);
    } finally {
      setLoading(false);
    }
  };

  if (loading) return <div className="loading">Loading...</div>;

  return (
    <div className="message-list">
      {messages.map((msg) => (
        <div key={msg.id} className="message">
          <span className="sender">User {msg.sender_id}</span>
          <p className="content">{msg.content}</p>
          <span className="time">{new Date(msg.created_at).toLocaleTimeString()}</span>
        </div>
      ))}
    </div>
  );
};

export default MessageList;
```

### 2. Form Submission (Send Message)

```tsx
import React, { useState } from 'react';

interface Props {
  conversationId: number;
  currentUserId: number;
  onMessageSent: () => void;
}

const MessageInput: React.FC<Props> = ({ conversationId, currentUserId, onMessageSent }) => {
  const [text, setText] = useState('');
  const [sending, setSending] = useState(false);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    if (!text.trim() || sending) return;

    setSending(true);
    try {
      await fetch(`/api/conversations/${conversationId}/messages`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          content: text.trim(),
          sender_id: currentUserId,
        }),
      });
      setText('');
      onMessageSent(); // Refresh message list
    } catch (err) {
      console.error('Failed to send:', err);
    } finally {
      setSending(false);
    }
  };

  return (
    <form onSubmit={handleSubmit} className="message-input">
      <input
        type="text"
        value={text}
        onChange={(e) => setText(e.target.value)}
        placeholder="Type a message..."
        disabled={sending}
      />
      <button type="submit" disabled={!text.trim() || sending}>
        Send
      </button>
    </form>
  );
};

export default MessageInput;
```

### 3. WebSocket Real-Time Updates

```tsx
import { useEffect, useRef } from 'react';

function useWebSocket(url: string, onMessage: (data: any) => void) {
  const wsRef = useRef<WebSocket | null>(null);

  useEffect(() => {
    const ws = new WebSocket(url);
    wsRef.current = ws;

    ws.onmessage = (event) => {
      const data = JSON.parse(event.data);
      onMessage(data);
    };

    ws.onclose = () => {
      console.log('WebSocket disconnected');
    };

    return () => {
      ws.close();
    };
  }, [url]);

  const send = (data: any) => {
    if (wsRef.current?.readyState === WebSocket.OPEN) {
      wsRef.current.send(JSON.stringify(data));
    }
  };

  return { send };
}

// Usage in a component:
// const { send } = useWebSocket('ws://localhost:3001', (data) => {
//   if (data.type === 'new_message') {
//     setMessages(prev => [...prev, data.message]);
//   }
// });
```

### 4. Typing Indicator Component

```tsx
import React, { useState, useEffect } from 'react';

interface Props {
  conversationId: number;
  currentUserId: number;
}

const TypingIndicator: React.FC<Props> = ({ conversationId, currentUserId }) => {
  const [typingUsers, setTypingUsers] = useState<string[]>([]);

  useEffect(() => {
    // Listen for typing events via WebSocket or polling
    const interval = setInterval(async () => {
      const res = await fetch(`/api/conversations/${conversationId}/typing`);
      const data = await res.json();
      setTypingUsers(data.users.filter((u: any) => u.id !== currentUserId));
    }, 2000);

    return () => clearInterval(interval);
  }, [conversationId, currentUserId]);

  if (typingUsers.length === 0) return null;

  return (
    <div className="typing-indicator">
      {typingUsers.length === 1
        ? `${typingUsers[0]} is typing...`
        : `${typingUsers.length} people are typing...`}
    </div>
  );
};

export default TypingIndicator;
```

### 5. Emoji Reaction Button

```tsx
import React, { useState } from 'react';

interface Props {
  messageId: number;
  currentUserId: number;
  existingReactions: { emoji: string; count: number; userReacted: boolean }[];
}

const ReactionBar: React.FC<Props> = ({ messageId, currentUserId, existingReactions }) => {
  const [reactions, setReactions] = useState(existingReactions);
  const [showPicker, setShowPicker] = useState(false);

  const QUICK_EMOJIS = ['👍', '❤️', '😂', '🎉', '😮', '😢'];

  const toggleReaction = async (emoji: string) => {
    try {
      const res = await fetch(`/api/messages/${messageId}/reactions`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ emoji, user_id: currentUserId }),
      });
      const updated = await res.json();
      setReactions(updated.reactions);
    } catch (err) {
      console.error('Failed to toggle reaction:', err);
    }
    setShowPicker(false);
  };

  return (
    <div className="reaction-bar">
      {reactions.map((r) => (
        <button
          key={r.emoji}
          className={`reaction ${r.userReacted ? 'active' : ''}`}
          onClick={() => toggleReaction(r.emoji)}
        >
          {r.emoji} {r.count}
        </button>
      ))}
      <button className="add-reaction" onClick={() => setShowPicker(!showPicker)}>
        +
      </button>
      {showPicker && (
        <div className="emoji-picker">
          {QUICK_EMOJIS.map((emoji) => (
            <button key={emoji} onClick={() => toggleReaction(emoji)}>
              {emoji}
            </button>
          ))}
        </div>
      )}
    </div>
  );
};

export default ReactionBar;
```

---

## Backend Patterns You MUST Know

### Node.js + Express (Most Likely Backend)

#### 1. Basic REST Controller

```typescript
import express, { Request, Response } from 'express';
import { Pool } from 'pg';

const router = express.Router();
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// GET /api/conversations/:id/messages
router.get('/conversations/:id/messages', async (req: Request, res: Response) => {
  try {
    const { id } = req.params;
    const result = await pool.query(
      `SELECT m.*, u.display_name as sender_name
       FROM messages m
       JOIN users u ON u.id = m.sender_id
       WHERE m.conversation_id = $1
       ORDER BY m.created_at ASC`,
      [id]
    );
    res.json(result.rows);
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: 'Internal server error' });
  }
});

// POST /api/conversations/:id/messages
router.post('/conversations/:id/messages', async (req: Request, res: Response) => {
  try {
    const { id } = req.params;
    const { content, sender_id } = req.body;

    if (!content || !sender_id) {
      return res.status(400).json({ error: 'content and sender_id required' });
    }

    const result = await pool.query(
      `INSERT INTO messages (conversation_id, sender_id, content, created_at)
       VALUES ($1, $2, $3, NOW())
       RETURNING *`,
      [id, sender_id, content]
    );

    res.status(201).json(result.rows[0]);
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: 'Internal server error' });
  }
});

export default router;
```

#### 2. Reactions Endpoint (Toggle)

```typescript
// POST /api/messages/:id/reactions — Toggle a reaction
router.post('/messages/:id/reactions', async (req: Request, res: Response) => {
  try {
    const { id } = req.params;
    const { emoji, user_id } = req.body;

    // Check if already reacted
    const existing = await pool.query(
      `SELECT id FROM reactions WHERE message_id = $1 AND user_id = $2 AND emoji = $3`,
      [id, user_id, emoji]
    );

    if (existing.rows.length > 0) {
      // Remove reaction (toggle off)
      await pool.query(`DELETE FROM reactions WHERE id = $1`, [existing.rows[0].id]);
    } else {
      // Add reaction (toggle on)
      await pool.query(
        `INSERT INTO reactions (message_id, user_id, emoji, created_at)
         VALUES ($1, $2, $3, NOW())`,
        [id, user_id, emoji]
      );
    }

    // Return updated reactions for this message
    const reactions = await pool.query(
      `SELECT emoji, COUNT(*) as count,
              BOOL_OR(user_id = $2) as user_reacted
       FROM reactions
       WHERE message_id = $1
       GROUP BY emoji`,
      [id, user_id]
    );

    res.json({ reactions: reactions.rows });
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: 'Internal server error' });
  }
});
```

#### 3. WebSocket Server (Real-Time)

```typescript
import { Server } from 'socket.io';
import http from 'http';

const server = http.createServer(app);
const io = new Server(server, { cors: { origin: '*' } });

io.on('connection', (socket) => {
  console.log('Client connected:', socket.id);

  // Join a conversation room
  socket.on('join_conversation', (conversationId: string) => {
    socket.join(`conversation:${conversationId}`);
  });

  // Broadcast new message to conversation participants
  socket.on('send_message', (data: { conversationId: string; message: any }) => {
    io.to(`conversation:${data.conversationId}`).emit('new_message', data.message);
  });

  // Typing indicator
  socket.on('typing', (data: { conversationId: string; userName: string }) => {
    socket.to(`conversation:${data.conversationId}`).emit('user_typing', {
      user: data.userName,
    });
  });

  socket.on('disconnect', () => {
    console.log('Client disconnected:', socket.id);
  });
});
```

#### 4. Conversation List with Unread Count

```typescript
// GET /api/users/:userId/conversations
router.get('/users/:userId/conversations', async (req: Request, res: Response) => {
  try {
    const { userId } = req.params;
    const result = await pool.query(
      `SELECT c.id, c.title, c.updated_at,
              (SELECT content FROM messages 
               WHERE conversation_id = c.id 
               ORDER BY created_at DESC LIMIT 1) as last_message,
              (SELECT COUNT(*) FROM messages m
               WHERE m.conversation_id = c.id
               AND m.created_at > COALESCE(
                 (SELECT last_read_at FROM participants 
                  WHERE conversation_id = c.id AND user_id = $1), 
                 '1970-01-01'
               )) as unread_count
       FROM conversations c
       JOIN participants p ON p.conversation_id = c.id
       WHERE p.user_id = $1
       ORDER BY c.updated_at DESC`,
      [userId]
    );
    res.json(result.rows);
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: 'Internal server error' });
  }
});
```

### Ruby on Rails (Alternative Backend)

#### Quick Reference (If They Offer Rails)

```ruby
# app/controllers/api/messages_controller.rb
class Api::MessagesController < ApplicationController
  def index
    conversation = Conversation.find(params[:conversation_id])
    messages = conversation.messages.includes(:sender).order(:created_at)
    render json: messages.as_json(include: { sender: { only: [:id, :display_name] } })
  end

  def create
    conversation = Conversation.find(params[:conversation_id])
    message = conversation.messages.build(message_params)

    if message.save
      # Broadcast via ActionCable (Rails WebSocket)
      ConversationChannel.broadcast_to(conversation, message.as_json)
      render json: message, status: :created
    else
      render json: { errors: message.errors.full_messages }, status: :unprocessable_entity
    end
  end

  private

  def message_params
    params.require(:message).permit(:content, :sender_id)
  end
end

# app/models/message.rb
class Message < ApplicationRecord
  belongs_to :conversation
  belongs_to :sender, class_name: 'User'
  has_many :reactions, dependent: :destroy

  validates :content, presence: true
end
```

---

## Practice Project: Mini Chat App

### Folder Structure (What You'll Likely See)

```
minicom/
├── frontend/
│   ├── src/
│   │   ├── components/
│   │   │   ├── App.tsx
│   │   │   ├── ConversationList.tsx
│   │   │   ├── MessageList.tsx
│   │   │   ├── MessageInput.tsx
│   │   │   └── Header.tsx
│   │   ├── types.ts
│   │   ├── api.ts
│   │   └── index.tsx
│   ├── package.json
│   └── tsconfig.json
├── backend/
│   ├── src/
│   │   ├── routes/
│   │   │   ├── conversations.ts
│   │   │   ├── messages.ts
│   │   │   └── users.ts
│   │   ├── db.ts
│   │   ├── app.ts
│   │   └── server.ts
│   ├── migrations/
│   ├── package.json
│   └── tsconfig.json
├── docker-compose.yml
└── README.md
```

### Your First 10 Minutes Checklist

```
1. Read README.md (setup instructions, existing features)
2. Run: docker-compose up (or npm install && npm start)
3. Open browser → click through existing UI
4. Identify: What components exist? What endpoints exist?
5. Find: Where are messages stored? What's the DB schema?
6. Plan: Which files do I need to modify?
```

---

## Common Minicom Tasks & Solutions

### Task 1: "Add customer-to-admin chat window"

**What they want:** Customer can send messages that appear in admin's inbox.

**Steps:**
1. Backend: Create endpoint `POST /api/conversations` (create new conversation)
2. Backend: Ensure messages endpoint works for new conversation
3. Frontend: Add "New Conversation" button
4. Frontend: Message input sends to correct conversation

### Task 2: "Add emoji reactions to messages"

**Steps:**
1. Backend: Create `POST /api/messages/:id/reactions` (toggle endpoint)
2. Backend: Include reactions when fetching messages
3. Frontend: Add reaction button below each message
4. Frontend: Show existing reactions with counts

### Task 3: "Add read receipts"

**Steps:**
1. Backend: `PUT /api/conversations/:id/read` — updates `last_read_at` for user
2. Backend: Return `unread_count` with conversation list
3. Frontend: Call read endpoint when user opens conversation
4. Frontend: Show unread badge on conversation list

### Task 4: "Add typing indicator"

**Steps:**
1. Backend: WebSocket event `typing` (broadcast to conversation room)
2. Frontend: Emit typing event on keypress (debounced)
3. Frontend: Show "X is typing..." when receiving event
4. Auto-hide after 3 seconds of no typing

### Task 5: "Add message search"

**Steps:**
1. Backend: `GET /api/messages/search?q=text` — full-text search
2. Frontend: Search input above conversation list
3. Frontend: Show matching messages with conversation context

---

## Pairing Session Strategy

### The 45 minutes pairing is where you WIN or LOSE

**What the interviewer expects:**
- You drive the keyboard initially
- You discuss trade-offs out loud
- You ask "What do you think?" or "Should we handle X?"
- You're receptive to suggestions
- You refactor if they hint at something

### Communication Templates

```
Starting a feature:
"I'm thinking we need to touch three things:
 1. The database/API to store [X]
 2. A new endpoint to [Y]
 3. The frontend component to [Z]
 Shall I start with the backend?"

When stuck:
"I'm not sure about the best approach here. 
 Option A would be [X], which is simpler but [tradeoff].
 Option B would be [Y], which handles [edge case] better.
 What do you think?"

After implementing:
"So this handles the basic case. 
 If we had more time, I'd add [error handling / validation / tests].
 Want me to extend this or move to [next feature]?"
```

### DO's and DON'Ts

| ✅ DO | ❌ DON'T |
|-------|---------|
| Think aloud | Code silently for 10+ min |
| Ask for opinions | Ignore interviewer's suggestions |
| Ship working code | Spend 20 min perfecting CSS |
| Handle basic errors | Add comprehensive error handling |
| Use console.log to debug | Pretend everything works first try |
| Admit unfamiliarity | Fake knowledge of a library |

---

## Time Management Blueprint

### Solo Phase (45 min)

```
┌─────────────────────────────────────────────┐
│ 0:00 - 0:10  EXPLORE                        │
│   • Read README                             │
│   • Run the app                             │
│   • Identify existing components/routes     │
│                                             │
│ 0:10 - 0:15  PLAN                           │
│   • Identify files to modify                │
│   • Mental model of data flow               │
│   • Decide: start backend or frontend?      │
│                                             │
│ 0:15 - 0:30  IMPLEMENT BACKEND              │
│   • New route/endpoint                      │
│   • DB query                                │
│   • Test with curl/Postman                  │
│                                             │
│ 0:30 - 0:45  IMPLEMENT FRONTEND             │
│   • Connect to new endpoint                 │
│   • Basic UI (functional, not pretty)       │
│   • Verify in browser                       │
└─────────────────────────────────────────────┘
```

### Pairing Phase (45 min)

```
┌─────────────────────────────────────────────┐
│ 0:45 - 0:50  DEMO what you built            │
│   • Show the working feature                │
│   • Explain your approach                   │
│                                             │
│ 0:50 - 1:05  EXTEND (interviewer guides)    │
│   • They suggest a new requirement          │
│   • Discuss approach together               │
│   • Implement while communicating           │
│                                             │
│ 1:05 - 1:25  BUILD MORE                     │
│   • Continue extending                      │
│   • Handle edge cases they mention          │
│   • Refactor if they suggest it             │
│                                             │
│ 1:25 - 1:30  WRAP UP                        │
│   • Summarize what was built                │
│   • Mention what you'd do next              │
└─────────────────────────────────────────────┘
```

---

## Quick TypeScript Types Reference

```typescript
// types.ts — Common types for Minicom

interface User {
  id: number;
  display_name: string;
  email: string;
  role: 'admin' | 'customer';
  avatar_url?: string;
  created_at: string;
}

interface Conversation {
  id: number;
  title?: string;
  created_at: string;
  updated_at: string;
  last_message?: string;
  unread_count?: number;
  participants: User[];
}

interface Message {
  id: number;
  conversation_id: number;
  sender_id: number;
  sender_name?: string;
  content: string;
  created_at: string;
  reactions?: Reaction[];
}

interface Reaction {
  emoji: string;
  count: number;
  user_reacted: boolean;
}

// API helper
const API_BASE = '/api';

async function apiGet<T>(path: string): Promise<T> {
  const res = await fetch(`${API_BASE}${path}`);
  if (!res.ok) throw new Error(`API error: ${res.status}`);
  return res.json();
}

async function apiPost<T>(path: string, body: any): Promise<T> {
  const res = await fetch(`${API_BASE}${path}`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(body),
  });
  if (!res.ok) throw new Error(`API error: ${res.status}`);
  return res.json();
}
```

---

## Practice Drill (Do This Before Interview)

**Timer: 45 minutes. Build a chat feature from scratch:**

1. Create a React component that shows a list of messages
2. Add a text input to send a new message (POST to API)
3. Add a "reaction" button that calls an endpoint
4. Make the UI auto-refresh every 3 seconds (polling)

**If you can do this in 45 min, you'll ace the Minicom round.**

---

*Previous: [← 12 SkillBasedRouting](./12_SkillBasedRouting.md) | Next: [14 ValuesRound BehavioralPrep →](./14_ValuesRound_BehavioralPrep.md)*
