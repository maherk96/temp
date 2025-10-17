```java
public static String format(SessionID sessionId, Message message, boolean inbound) {
    var fix = sanitize(message.toString());

    var senderCompID = safeGet(message, quickfix.field.SenderCompID.FIELD);
    var targetCompID = safeGet(message, quickfix.field.TargetCompID.FIELD);

    var msgDesc = getMessageDescription(message);
    var session = sessionId != null ? sessionId.toString() : "unknown-session";

    var arrow = inbound ? "<----" : "---->";
    var dirTag = inbound ? "IN >>>" : "OUT <<<";

    var clOrdID = safeGet(message, quickfix.field.ClOrdID.FIELD);
    var clOrdLinkID = safeGet(message, quickfix.field.ClOrdLinkID.FIELD);
    var execType = safeGet(message, quickfix.field.ExecType.FIELD);

    var clientName = inbound ? targetCompID : senderCompID;

    var msgInfo = new StringBuilder();
    if (clOrdID != null && !clOrdID.isEmpty() && !"unknown".equals(clOrdID)) {
        msgInfo.append("ClOrdID:").append(clOrdID);
    }
    if (clOrdLinkID != null && !clOrdLinkID.isEmpty() && !"unknown".equals(clOrdLinkID)) {
        if (msgInfo.length() > 0) msgInfo.append("|");
        msgInfo.append("ClOrdLinkID:").append(clOrdLinkID);
    }
    if (execType != null && !execType.isEmpty() && !"unknown".equals(execType)) {
        if (msgInfo.length() > 0) msgInfo.append("|");
        msgInfo.append("ExecType:").append(getExecTypeDescription(execType));
    }

    var msgStr = msgInfo.length() > 0 ? "[" + msgInfo.toString().trim() + "]" : "";

    String directionStr = inbound
        ? String.format("[%s %s %s]", senderCompID, arrow, targetCompID)   // COR ----> QA1 (inbound)
        : String.format("[%s %s %s]", senderCompID, arrow, targetCompID);  // QA1 ----> COR (outbound)

    return String.format(
        "[%s][session=%s]%n%s %s %s %s%nFIX: %s",
        clientName, session, dirTag, msgDesc, msgStr, directionStr, fix
    );
}

```
