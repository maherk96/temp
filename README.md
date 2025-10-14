```java

/**
 * Returns the human-readable description for a given FIX ExecType code.
 * 
 * @param execTypeChar The single-character ExecType (e.g., "0", "A", "F").
 * @return A descriptive string for the execution type.
 */
public static String getExecTypeDescription(String execTypeChar) {
    if (execTypeChar == null || execTypeChar.isEmpty()) {
        return "Unknown ExecType";
    }
    
    return switch (execTypeChar) {
        case "0" -> "New";
        case "1" -> "Partial Fill";
        case "2" -> "Fill";
        case "3" -> "Done For Day";
        case "4" -> "Canceled";
        case "5" -> "Replace";
        case "6" -> "Pending Cancel";
        case "7" -> "Stopped";
        case "8" -> "Rejected";
        case "9" -> "Suspended";
        case "A" -> "Pending New";
        case "B" -> "Calculated";
        case "C" -> "Expired";
        case "D" -> "Restated";
        case "E" -> "Pending Replace";
        case "F" -> "Trade";
        case "I" -> "Order Status";
        default -> "Unknown ExecType (" + execTypeChar + ")";
    };
}
```
