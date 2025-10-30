# Custom Frontend

Parlant를 React 애플리케이션에 통합하는 가장 빠른 방법은 공식 [`parlant-chat-react`](https://github.com/emcie-co/parlant-chat-react) 위젯을 사용하는 것입니다. 이 컴포넌트는 Parlant 에이전트에 직접 연결되는 완전한 채팅 인터페이스를 제공합니다.

### 설치 및 기본 설정

npm 또는 yarn을 통해 위젯을 설치합니다:

```bash
npm install parlant-chat-react
# 또는
yarn add parlant-chat-react
```

그런 다음 React 애플리케이션에 통합합니다:

```jsx
import React from 'react';
import ParlantChatbox from 'parlant-chat-react';

function App() {
  return (
    <div>
      <h1>My Application</h1>
      <ParlantChatbox
        server="http://localhost:8800"  // Parlant 서버 URL
        agentId="your-agent-id"         // 에이전트 ID
      />
    </div>
  );
}

export default App;
```

### 구성 옵션

위젯은 여러 구성 props를 지원합니다:

```jsx
<ParlantChatbox
  // 필수 props
  server="http://localhost:8800"
  agentId="your-agent-id"

  // 선택적 props
  sessionId="existing-session-id"     // 기존 세션 계속
  customerId="customer-123"           // 특정 고객과 연결
  float={true}                        // 플로팅 팝업으로 표시
  titleFn={(session) => `Chat ${session.id}`}  // 동적 제목 생성
/>
```

### 일반적인 커스터마이제이션

#### 맞춤형 클래스로 스타일링

CSS 클래스 재정의를 사용하여 외관을 커스터마이징합니다:

```jsx
<ParlantChatbox
  server="http://localhost:8800"
  agentId="your-agent-id"
  classNames={{
    chatboxWrapper: "my-chat-wrapper",
    chatbox: "my-chatbox",
    messagesArea: "my-messages",
    agentMessage: "my-agent-bubble",
    customerMessage: "my-customer-bubble",
    textarea: "my-input-field",
    popupButton: "my-popup-btn"
  }}
/>
```

#### 맞춤형 컴포넌트 교체

자체 컴포넌트로 특정 컴포넌트를 교체합니다:

```jsx
<ParlantChatbox
  server="http://localhost:8800"
  agentId="your-agent-id"
  components={{
    popupButton: ({ toggleChatOpen }) => (
      <button
        onClick={toggleChatOpen}
        className="custom-chat-button"
      >
        💬 Chat with us
      </button>
    ),
    agentMessage: ({ message }) => (
      <div className="custom-agent-message">
        <img src="https://parlant.io/agent-avatar.png" alt="Agent" />
        <p>{message.data.message}</p>
      </div>
    )
  }}
/>
```

#### 플로팅 채팅 모드

플로팅 채팅 인터페이스를 위해 팝업 모드를 활성화합니다:

```jsx
<ParlantChatbox
  server="http://localhost:8800"
  agentId="your-agent-id"
  float={true}
  popupButton={<ChatIcon size={24} color="white" />}
/>
```

> **참조 구현**
>
> parlant-chat-react 위젯은 오픈 소스입니다! Vue, Angular 또는 vanilla JavaScript와 같은 다른 UI 프레임워크에서 맞춤형 위젯을 생성하기 위한 참조로 [GitHub에서 구현을 검토](https://github.com/emcie-co/parlant-chat-react)할 수 있습니다. 소스 코드는 세션 관리, 이벤트 처리 및 UI 상태 동기화에 대한 모범 사례를 보여줍니다.

## 맞춤형 프론트엔드 구축

React 위젯이 제공하는 것보다 더 많은 제어가 필요하거나 다른 프레임워크를 사용하는 경우, Parlant의 클라이언트 API를 직접 사용하여 맞춤형 프론트엔드를 구축할 수 있습니다.

### 1단계: Parlant 클라이언트 초기화

서버와 통신하기 위해 Parlant 클라이언트 설정을 시작합니다:

#### TypeScript

```typescript
import { ParlantClient } from 'parlant-client';

class ParlantChat {
  private client: ParlantClient;
  private sessionId: string | null = null;
  private lastOffset: number = 0;

  constructor(serverUrl: string) {
    this.client = new ParlantClient({
      environment: serverUrl
    });
  }
}
```

#### JavaScript

```javascript
import { ParlantClient } from 'parlant-client';

class ParlantChat {
  constructor(serverUrl) {
    this.client = new ParlantClient({
      environment: serverUrl
    });
    this.sessionId = null;
    this.lastOffset = 0;
  }
}
```

### 2단계: 세션 생성

에이전트와의 대화 세션을 초기화합니다:

#### TypeScript

```typescript
async createSession(agentId: string, customerId?: string): Promise<string> {
  try {
    const session = await this.client.sessions.create({
      agentId: agentId,
      customerId: customerId,
      title: `Chat Session ${new Date().toLocaleString()}`
    });

    this.sessionId = session.id;
    console.log('Session created:', this.sessionId);

    // 이벤트 모니터링 시작
    this.startEventMonitoring();

    return this.sessionId;
  } catch (error) {
    console.error('Failed to create session:', error);
    throw error;
  }
}
```

#### JavaScript

```javascript
async createSession(agentId, customerId) {
  try {
    const session = await this.client.sessions.create({
      agentId: agentId,
      customerId: customerId,
      title: `Chat Session ${new Date().toLocaleString()}`
    });

    this.sessionId = session.id;
    console.log('Session created:', this.sessionId);

    // 이벤트 모니터링 시작
    this.startEventMonitoring();

    return this.sessionId;
  } catch (error) {
    console.error('Failed to create session:', error);
    throw error;
  }
}
```

### 3단계: 고객 메시지 전송

사용자 입력을 처리하고 에이전트에게 메시지를 전송합니다:

#### TypeScript

```typescript
async sendMessage(message: string): Promise<void> {
  if (!this.sessionId) {
    throw new Error('No active session');
  }

  try {
    await this.client.sessions.createEvent(this.sessionId, {
      kind: "message",
      source: "customer",
      message: message
    });

    // 메시지는 이벤트 모니터링에서 돌아올 때 UI에 나타납니다
    console.log('Message sent:', message);
  } catch (error) {
    console.error('Failed to send message:', error);
    throw error;
  }
}
```

#### JavaScript

```javascript
async sendMessage(message) {
  if (!this.sessionId) {
    throw new Error('No active session');
  }

  try {
    await this.client.sessions.createEvent(this.sessionId, {
      kind: "message",
      source: "customer",
      message: message
    });

    // 메시지는 이벤트 모니터링에서 돌아올 때 UI에 나타납니다
    console.log('Message sent:', message);
  } catch (error) {
    console.error('Failed to send message:', error);
    throw error;
  }
}
```

### 4단계: 세션 이벤트 모니터링

메시지와 업데이트를 받기 위한 이벤트 모니터링을 구현합니다:

#### TypeScript

```typescript
private async startEventMonitoring(): Promise<void> {
  if (!this.sessionId) return;

  while (true) {
    try {
      // Long polling으로 새 이벤트 폴링
      const events = await this.client.sessions.listEvents(this.sessionId, {
        minOffset: this.lastOffset,
        waitForData: 30, // 최대 30초 동안 새 이벤트 대기
        kinds: ["message", "status"] // 메시지 및 상태 이벤트만 가져오기
      });

      // 각 이벤트 처리
      for (const event of events) {
        await this.handleEvent(event);
        this.lastOffset = Math.max(this.lastOffset, event.offset + 1);
      }

    } catch (error) {
      console.error('Event monitoring error:', error);
      // 재시도 전 대기
      await new Promise(resolve => setTimeout(resolve, 5000));
    }
  }
}

private async handleEvent(event: any): Promise<void> {
  if (event.kind === "message") {
    this.displayMessage(event);
  } else if (event.kind === "status") {
    this.updateStatus(event.data.status);
  }
}
```

#### JavaScript

```javascript
async startEventMonitoring() {
  if (!this.sessionId) return;

  while (true) {
    try {
      // Long polling으로 새 이벤트 폴링
      const events = await this.client.sessions.listEvents(this.sessionId, {
        minOffset: this.lastOffset,
        waitForData: 30, // 최대 30초 동안 새 이벤트 대기
        kinds: ["message", "status"] // 메시지 및 상태 이벤트만 가져오기
      });

      // 각 이벤트 처리
      for (const event of events) {
        await this.handleEvent(event);
        this.lastOffset = Math.max(this.lastOffset, event.offset + 1);
      }

    } catch (error) {
      console.error('Event monitoring error:', error);
      // 재시도 전 대기
      await new Promise(resolve => setTimeout(resolve, 5000));
    }
  }
}

async handleEvent(event) {
  if (event.kind === "message") {
    this.displayMessage(event);
  } else if (event.kind === "status") {
    this.updateStatus(event.data.status);
  }
}
```

### 5단계: UI에 메시지 표시

Parlant의 이벤트를 기반으로 UI 업데이트를 구현합니다:

#### TypeScript

```typescript
private displayMessage(event: any): void {
  const messageElement = document.createElement('div');
  messageElement.className = `message ${event.source}`;

  // 메시지 소스에 따라 스타일 지정
  switch (event.source) {
    case 'customer':
      messageElement.classList.add('customer-message');
      break;
    case 'ai_agent':
      messageElement.classList.add('agent-message');
      break;
    case 'human_agent':
      messageElement.classList.add('human-agent-message');
      const agentName = event.data.participant?.display_name || 'Agent';
      messageElement.innerHTML = `
        <div class="agent-info">${agentName}</div>
        <div class="message-content">${event.data.message}</div>
      `;
      break;
  }

  // 채팅 컨테이너에 추가
  const chatContainer = document.getElementById('chat-messages');
  if (chatContainer) {
    chatContainer.appendChild(messageElement);
    chatContainer.scrollTop = chatContainer.scrollHeight;
  }
}

private updateStatus(status: string): void {
  const statusElement = document.getElementById('chat-status');
  if (statusElement) {
    switch (status) {
      case 'processing':
        statusElement.textContent = 'Agent is thinking...';
        break;
      case 'typing':
        statusElement.textContent = 'Agent is typing...';
        break;
      case 'ready':
        statusElement.textContent = '';
        break;
    }
  }
}
```

#### JavaScript

```javascript
displayMessage(event) {
  const messageElement = document.createElement('div');
  messageElement.className = `message ${event.source}`;

  // 메시지 소스에 따라 스타일 지정
  switch (event.source) {
    case 'customer':
      messageElement.classList.add('customer-message');
      messageElement.innerHTML = `<div class="message-content">${event.data.message}</div>`;
      break;
    case 'ai_agent':
      messageElement.classList.add('agent-message');
      messageElement.innerHTML = `<div class="message-content">${event.data.message}</div>`;
      break;
    case 'human_agent':
      messageElement.classList.add('human-agent-message');
      const agentName = event.data.participant?.display_name || 'Agent';
      messageElement.innerHTML = `
        <div class="agent-info">${agentName}</div>
        <div class="message-content">${event.data.message}</div>
      `;
      break;
  }

  // 채팅 컨테이너에 추가
  const chatContainer = document.getElementById('chat-messages');
  if (chatContainer) {
    chatContainer.appendChild(messageElement);
    chatContainer.scrollTop = chatContainer.scrollHeight;
  }
}

updateStatus(status) {
  const statusElement = document.getElementById('chat-status');
  if (statusElement) {
    switch (status) {
      case 'processing':
        statusElement.textContent = 'Agent is thinking...';
        break;
      case 'typing':
        statusElement.textContent = 'Agent is typing...';
        break;
      case 'ready':
        statusElement.textContent = '';
        break;
    }
  }
}
```

### 6단계: 완전한 HTML 예제

맞춤형 구현을 보여주는 완전한 HTML 페이지는 다음과 같습니다:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Custom Parlant Chat</title>
    <style>
        .chat-container {
            max-width: 500px;
            margin: 50px auto;
            border: 1px solid #ddd;
            border-radius: 8px;
            overflow: hidden;
        }

        .chat-header {
            background: #007bff;
            color: white;
            padding: 15px;
            text-align: center;
        }

        .chat-messages {
            height: 400px;
            padding: 15px;
            overflow-y: auto;
            background: #f8f9fa;
        }

        .message {
            margin: 10px 0;
            padding: 10px;
            border-radius: 8px;
            max-width: 80%;
        }

        .customer-message {
            background: #007bff;
            color: white;
            margin-left: auto;
            text-align: right;
        }

        .agent-message {
            background: white;
            border: 1px solid #ddd;
        }

        .human-agent-message {
            background: #28a745;
            color: white;
        }

        .chat-input {
            display: flex;
            padding: 15px;
            background: white;
        }

        .chat-input input {
            flex: 1;
            padding: 10px;
            border: 1px solid #ddd;
            border-radius: 4px;
            margin-right: 10px;
        }

        .chat-input button {
            padding: 10px 20px;
            background: #007bff;
            color: white;
            border: none;
            border-radius: 4px;
            cursor: pointer;
        }

        #chat-status {
            font-style: italic;
            color: #666;
            padding: 5px 15px;
        }
    </style>
</head>
<body>
    <div class="chat-container">
        <div class="chat-header">
            <h3>Customer Support Chat</h3>
        </div>
        <div id="chat-status"></div>
        <div id="chat-messages" class="chat-messages"></div>
        <div class="chat-input">
            <input
                type="text"
                id="message-input"
                placeholder="Type your message..."
                onkeypress="handleKeyPress(event)"
            />
            <button onclick="sendMessage()">Send</button>
        </div>
    </div>

    <script type="module">
        import { ParlantClient } from 'https://unpkg.com/parlant-client@latest/dist/index.js';

        // 맞춤형 채팅 초기화
        const chat = new ParlantChat('http://localhost:8800');

        // 채팅 세션 시작
        chat.createSession('your-agent-id')
            .then(sessionId => {
                console.log('Chat ready!', sessionId);
            })
            .catch(error => {
                console.error('Failed to start chat:', error);
            });

        // 함수를 전역적으로 사용 가능하게 만들기
        window.sendMessage = () => chat.sendUserMessage();
        window.handleKeyPress = (event) => {
            if (event.key === 'Enter') {
                chat.sendUserMessage();
            }
        };
    </script>
</body>
</html>
```

### 주요 구현 원칙

1. **이벤트 기반 아키텍처**: 채팅은 Parlant 세션의 이벤트에 의해 구동되어 서버 상태와의 일관성을 보장합니다.
2. **Long Polling**: `listEvents()`에서 `waitForData` 매개변수를 사용하여 지속적인 폴링 없이 효율적인 실시간 업데이트를 수행합니다.
3. **상태 동기화**: 항상 UI를 낙관적으로 업데이트하는 것이 아니라 Parlant 이벤트에서 오는 것을 표시합니다.
4. **오류 처리**: 네트워크 문제에 대한 강력한 오류 처리 및 재시도 로직을 구현합니다.
5. **반응형 디자인**: 채팅 인터페이스가 데스크톱과 모바일 장치 모두에서 잘 작동하도록 보장합니다.

이 접근 방식은 Parlant의 강력한 에이전트 기능을 활용하면서 채팅 경험에 대한 완전한 제어를 제공합니다. 이 패턴을 모든 프론트엔드 프레임워크 또는 vanilla JavaScript 구현에 적용할 수 있습니다.
