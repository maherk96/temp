```java

import lombok.extern.slf4j.Slf4j;
import quickfix.Message;
import quickfix.SessionID;

@Slf4j
public class AlgoFixMessageListener implements QFMessageListener {

    private final AlgoFixMessageQueue algoFixMessageQueue;

    public AlgoFixMessageListener(AlgoFixMessageQueue algoFixMessageQueue) {
        this.algoFixMessageQueue = algoFixMessageQueue;
    }

    @Override
    public void onMessage(SessionID sessionId, Message message) {
        try {
            // Here it's inbound (server → client)
            String formatted = FixLogFormatter.format(sessionId, message, true);
            log.info(formatted);

            algoFixMessageQueue.addMessage(message.toString());
        } catch (Exception e) {
            log.warn("Failed to extract FIX message details: {}", e.getMessage());
        }
    }
}

```
