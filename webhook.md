# Webhook

- 웹훅은 API 제공자에게 어떤 이벤트가 발생했을 때 API 제공자가 API 사용자에게 데이터를 보내주는 과정이나 기능을 의미한다.
- 일반적인 API는 API 사용자의 요청에 따라 수동적으로 응답을 반환하지만,
- 웹흑은 API 제공자가 능동적으로 API 사용자에게 데이터를 보낸다는 점에서 일반적인 API와 다르다.
- API 제공자가 API 사용자에게 데이터를 보내려면 API 사용자 쪽 엔드포인트를 알아야 한다.
- API 사용자는 이메일이나 폼 양식, 또는 별도의 API 등 API 제공자가 제공하는 수단에 맞게 API 사용자 쪽 엔드포인트(callback URL)와 관심있는 이벤트를 API 제공자에게 등록해야 한다.
- OpenAPI 3.1.0에 추가된 `webhooks`에는 API 제공자가 API 사용자에게 보내줄 데이터 규격과 API 사용자로부터 받을 상태 코드를 정의해서 API 사용자에게 알려줄 뿐 호출의 대상이 아니다.
  - OpenAPI 3.1을 사용할 수 있는 https://editor-next.swagger.io/ 에서도 `webhooks` 항목에는 `Try it out` 버튼이 없다.

## 웹훅과 서버-to-서버 API의 차이

Webhooks and server-to-server APIs are both mechanisms for communication between different software systems, but they serve different purposes and have distinct characteristics:

### Webhooks

- Event-Driven Communication: Webhooks are designed to notify external systems (subscribers) about specific events that occur in the API provider's system. These events could be things like resource updates, creations, or deletions.

- Push Model: In a webhook setup, the API provider initiates communication by sending HTTP requests to registered subscriber endpoints whenever an event occurs. This means that the subscriber's system needs to be capable of receiving incoming HTTP requests.

- Subscriber-Controlled: Subscribers are responsible for registering their own endpoints with the API provider. They specify the events they are interested in and provide the URL where they want to receive notifications.

- Asynchronous: Webhook notifications are typically asynchronous. The API provider sends notifications to subscribers without waiting for an immediate response. This can help decouple systems and improve scalability.

- Usage Scenarios: Webhooks are commonly used for scenarios where real-time or near-real-time notifications are needed. For example, notifying subscribers about order updates, new content availability, or social media interactions.

### Server-to-Server APIs:

- Data Exchange: Server-to-server APIs involve requesting and exchanging data between systems. One system (the client) makes API requests to another system (the server) to retrieve or send data.

- Pull Model: In a server-to-server API scenario, the client initiates communication by sending requests to the server's endpoints. The server responds to these requests with the requested data.

- Client-Controlled: The client decides when to make API requests and what data to retrieve or send. The server doesn't initiate communication with the client; it only responds to client requests.

- Synchronous: Server-to-server API interactions are typically synchronous. The client sends a request and waits for the server's response, which contains the requested data or an acknowledgment of the action.

- Usage Scenarios: Server-to-server APIs are used for scenarios where data needs to be fetched, updated, or synchronized between systems. Examples include retrieving user data, posting content, or querying a database.

In summary, the main difference between webhooks and server-to-server APIs lies in their purpose and communication model. Webhooks are designed for event-driven notifications initiated by the API provider, while server-to-server APIs involve data exchange initiated by the client. The choice between these two approaches depends on the specific use case and the type of interaction required between the systems.


