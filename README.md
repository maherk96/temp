```java
package com.citi.fx.qa.cor.mock.client.sim.util;

import quickfix.Message;
import quickfix.SessionID;

public class FixLogFormatter {

    public static String format(SessionID sessionId, Message message, boolean inbound) {
        String fix = sanitize(message.toString());

        // Extract comp IDs
        String senderCompID = safeGet(message, quickfix.field.SenderCompID.FIELD);
        String targetCompID = safeGet(message, quickfix.field.TargetCompID.FIELD);

        String msgDesc = getMessageDescription(message);
        String session = sessionId != null ? sessionId.toString() : "unknown-session";

        // Determine direction
        String arrow = inbound ? "<----" : "---->";
        String dirTag = inbound ? "IN  >>>" : "OUT <<<";

        // Derive "client name" dynamically — from the TargetCompID if inbound, SenderCompID if outbound
        String clientName = inbound ? targetCompID : senderCompID;

        return String.format(
            "[%s][session=%s]%n%s %s [%s %s %s]%nFIX (short): %s",
            clientName,
            session,
            dirTag,
            msgDesc,
            targetCompID,
            arrow,
            senderCompID,
            fix
        );
    }

    private static String sanitize(String rawFix) {
        return rawFix.replace('\u0001', '|');
    }

    private static String safeGet(Message message, int tag) {
        try {
            return message.getHeader().getString(tag);
        } catch (Exception e) {
            return "unknown";
        }
    }

    private static String getMessageDescription(Message message) {
        try {
            String msgType = message.getHeader().getString(quickfix.field.MsgType.FIELD);
            return com.citi.fx.qa.cor.mock.client.sim.listeners.FixMessageUtil.getMessageDescription(msgType);
        } catch (Exception e) {
            return "Unknown MsgType";
        }
    }
}


```
