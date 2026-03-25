```java

package com.mk.fx.qa.qap.logging.log4j2;

import com.mk.fx.qa.qap.logging.core.QAPLogCaptureConfig;
import com.mk.fx.qa.qap.logging.core.QAPLogCapturer;
import com.mk.fx.qa.qap.logging.core.QAPLogEntry;
import java.util.ArrayList;
import java.util.Collection;
import java.util.Collections;
import java.util.List;
import java.util.Objects;
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.core.Logger;
import org.apache.logging.log4j.core.LoggerContext;
import org.slf4j.LoggerFactory;

/**
 * Log4j2 implementation of QAPLogCapturer. Captures logs from Log4j2 by attaching a custom appender
 * to all configured loggers.
 *
 * <p>This implementation attaches to ALL loggers (not just root) to ensure logs from loggers with
 * {@code additivity="false"} are also captured.
 *
 * <p>Thread-safe and designed for parallel test execution.
 */
public class Log4j2Capturer implements QAPLogCapturer {

  private static final org.slf4j.Logger log = LoggerFactory.getLogger(Log4j2Capturer.class);
  private static final String APPENDER_NAME = "QAPLog4j2Appender";

  private volatile QAPLog4j2Appender appender;
  private volatile boolean initialized = false;
  
  // Track loggers we've attached to for proper cleanup
  private final List<Logger> attachedLoggers = Collections.synchronizedList(new ArrayList<>());

  @Override
  public void startCapture(String testId, QAPLogCaptureConfig config) {
    Objects.requireNonNull(testId, "testId cannot be null");
    Objects.requireNonNull(config, "config cannot be null");

    if (!config.isEnabled()) {
      log.debug("Log capture disabled for test: {}", testId);
      return;
    }

    ensureInitialized();

    if (appender != null) {
      appender.startCapture(testId, config);
      log.debug("Started Log4j2 capture for test: {}", testId);
    } else {
      log.warn("Failed to start capture - appender not initialized");
    }
  }

  @Override
  public List<QAPLogEntry> stopCapture(String testId) {
    Objects.requireNonNull(testId, "testId cannot be null");

    if (appender == null) {
      log.warn("Appender not initialized, returning empty log list for test: {}", testId);
      return Collections.emptyList();
    }

    List<QAPLogEntry> logs = appender.stopCapture(testId);
    log.debug("Stopped Log4j2 capture for test: {} ({} entries)", testId, logs.size());
    return logs;
  }

  @Override
  public String getFrameworkName() {
    return "Log4j2";
  }

  @Override
  public boolean isAvailable() {
    try {
      // Check if Log4j2 classes are on the classpath
      Class.forName("org.apache.logging.log4j.core.LoggerContext");
      Class.forName("org.apache.logging.log4j.core.appender.AbstractAppender");

      // Try to get the LoggerContext
      org.apache.logging.log4j.core.LoggerContext context =
          (org.apache.logging.log4j.core.LoggerContext) LogManager.getContext(false);
      if (context == null) {
        log.debug("Log4j2 LoggerContext is null");
        return false;
      }

      log.debug("Log4j2 is available and ready");
      return true;
    } catch (ClassNotFoundException e) {
      log.debug("Log4j2 classes not found on classpath: {}", e.getMessage());
      return false;
    } catch (Exception e) {
      log.warn("Error checking Log4j2 availability: {}", e.getMessage());
      return false;
    }
  }

  @Override
  public int getPriority() {
    return 100; // Higher priority than Logback (default 0)
  }

  @Override
  public void shutdown() {
    if (appender != null) {
      try {
        appender.cleanupThreadLocals();

        // Remove appender from all loggers we attached to
        for (Logger logger : attachedLoggers) {
          try {
            logger.removeAppender(appender);
          } catch (Exception e) {
            log.debug("Error removing appender from logger {}: {}", logger.getName(), e.getMessage());
          }
        }
        attachedLoggers.clear();
        
        appender.stop();
        log.info("QAP Log4j2 appender removed from {} loggers and stopped", attachedLoggers.size());
      } catch (Exception e) {
        log.warn("Error during Log4j2 capturer shutdown", e);
      } finally {
        appender = null;
        initialized = false;
      }
    }
  }

  /**
   * Initializes the appender and attaches it to ALL configured loggers. This ensures logs from
   * loggers with {@code additivity="false"} are captured.
   *
   * <p>This is done lazily on the first capture request.
   */
  private synchronized void ensureInitialized() {
    if (initialized) {
      return;
    }

    try {
      LoggerContext context = (LoggerContext) LogManager.getContext(false);

      // Create and start the appender
      appender = QAPLog4j2Appender.createAppender(APPENDER_NAME, null, true);
      if (appender == null) {
        throw new IllegalStateException("Failed to create QAPLog4j2Appender");
      }
      appender.start();

      // Attach to root logger
      Logger rootLogger = context.getRootLogger();
      attachAppenderToLogger(rootLogger);

      // Attach to ALL configured loggers to catch those with additivity=false
      Collection<Logger> loggers = context.getLoggers();
      int nonAdditiveCount = 0;
      for (Logger logger : loggers) {
        if (!logger.isAdditive()) {
          attachAppenderToLogger(logger);
          nonAdditiveCount++;
        }
      }

      initialized = true;
      log.info(
          "QAP Log4j2 appender attached to root logger and {} non-additive logger(s)",
          nonAdditiveCount);

    } catch (Exception e) {
      log.error("Failed to initialize Log4j2 capturer", e);
      throw new RuntimeException("Failed to initialize Log4j2 capturer", e);
    }
  }

  /**
   * Attaches the appender to a logger if not already attached.
   *
   * @param logger the logger to attach to
   */
  private void attachAppenderToLogger(Logger logger) {
    if (!logger.getAppenders().containsKey(APPENDER_NAME)) {
      logger.addAppender(appender);
      attachedLoggers.add(logger);
      log.debug("Attached QAP appender to logger: {}", logger.getName());
    }
  }

  /**
   * For testing: checks if the appender has any active captures.
   *
   * @return true if at least one test is actively capturing logs
   */
  boolean hasActiveCaptures() {
    return appender != null && appender.hasActiveCaptures();
  }
}








package com.mk.fx.qa.qap.logging.log4j2;

import com.mk.fx.qa.qap.logging.core.QAPLogCaptureConfig;
import com.mk.fx.qa.qap.logging.core.QAPLogEntry;
import com.mk.fx.qa.qap.logging.core.QAPLogLevel;
import java.time.Instant;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.CopyOnWriteArrayList;
import org.apache.logging.log4j.core.Appender;
import org.apache.logging.log4j.core.Core;
import org.apache.logging.log4j.core.Filter;
import org.apache.logging.log4j.core.LogEvent;
import org.apache.logging.log4j.core.appender.AbstractAppender;
import org.apache.logging.log4j.core.config.Property;
import org.apache.logging.log4j.core.config.plugins.Plugin;
import org.apache.logging.log4j.core.config.plugins.PluginAttribute;
import org.apache.logging.log4j.core.config.plugins.PluginElement;
import org.apache.logging.log4j.core.config.plugins.PluginFactory;
import org.apache.logging.log4j.message.Message;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

/**
 * Custom Log4j2 appender that captures log events in a thread-safe manner.
 *
 * <p>Logs are stored in a global ConcurrentHashMap keyed by testId so that events emitted from
 * <em>any</em> thread (main thread, test-worker, background threads, thread-pool threads, etc.) are
 * captured. Test isolation is enforced through unique testIds, not thread identity.
 *
 * <p>This appender is dynamically attached to the root logger when log capture starts and detached
 * when it stops.
 */
@Plugin(
    name = "QAPLog4j2Appender",
    category = Core.CATEGORY_NAME,
    elementType = Appender.ELEMENT_TYPE)
public class QAPLog4j2Appender extends AbstractAppender {

  private static final Logger log = LoggerFactory.getLogger(QAPLog4j2Appender.class);

  /**
   * Holds the capture state for a single test. Using a dedicated class ensures atomic access to
   * both the log buffer and the "max entries reached" flag.
   */
  private static class CaptureState {
    final CopyOnWriteArrayList<QAPLogEntry> logs = new CopyOnWriteArrayList<>();
    final QAPLogCaptureConfig config;
    volatile boolean maxEntriesWarningLogged = false;

    CaptureState(QAPLogCaptureConfig config) {
      this.config = config;
    }
  }

  // Global capture states keyed by testId - captures logs from ALL threads
  private final ConcurrentHashMap<String, CaptureState> captureStates = new ConcurrentHashMap<>();

  protected QAPLog4j2Appender(String name, Filter filter, boolean ignoreExceptions) {
    super(name, filter, null, ignoreExceptions, Property.EMPTY_ARRAY);
  }

  @PluginFactory
  public static QAPLog4j2Appender createAppender(
      @PluginAttribute("name") String name,
      @PluginElement("Filter") Filter filter,
      @PluginAttribute(value = "ignoreExceptions", defaultBoolean = true)
          boolean ignoreExceptions) {
    if (name == null) {
      LOGGER.error("No name provided for QAPLog4j2Appender");
      return null;
    }
    return new QAPLog4j2Appender(name, filter, ignoreExceptions);
  }

  /**
   * Starts capturing logs for a specific test. Logs produced by <em>any</em> thread will be
   * captured until {@link #stopCapture(String)} is called with the same testId.
   *
   * @param testId unique test identifier
   * @param config capture configuration
   */
  public void startCapture(String testId, QAPLogCaptureConfig config) {
    Objects.requireNonNull(testId, "testId cannot be null");
    Objects.requireNonNull(config, "config cannot be null");

    if (!config.isEnabled()) {
      log.debug("Log capture disabled for test: {}", testId);
      return;
    }

    captureStates.put(testId, new CaptureState(config));
    log.debug("Started log capture for test: {}", testId);
  }

  /**
   * Stops capturing logs and returns all captured entries, including those produced by threads other
   * than the one that called {@link #startCapture(String, QAPLogCaptureConfig)}.
   *
   * @param testId unique test identifier
   * @return list of captured log entries, never null
   */
  public List<QAPLogEntry> stopCapture(String testId) {
    Objects.requireNonNull(testId, "testId cannot be null");

    CaptureState state = captureStates.remove(testId);

    if (state == null) {
      log.warn("No active capture found for test: {}", testId);
      return Collections.emptyList();
    }

    // Return a defensive copy; CopyOnWriteArrayList's iterator is already a snapshot
    List<QAPLogEntry> result = new ArrayList<>(state.logs);
    log.debug("Stopped log capture for test: {} ({} entries)", testId, result.size());
    return result;
  }

  @Override
  public void append(LogEvent event) {
    if (captureStates.isEmpty()) {
      return; // No active captures, skip processing
    }

    try {
      // Snapshot the entry set to avoid ConcurrentModificationException
      for (Map.Entry<String, CaptureState> entry : captureStates.entrySet()) {
        String testId = entry.getKey();
        CaptureState state = entry.getValue();

        if (shouldCapture(event, state.config)) {
          QAPLogEntry logEntry = convertLogEvent(event, state.config);
          addLogEntry(testId, logEntry, state);
        }
      }
    } catch (Exception e) {
      // Never throw exceptions from appender - could break application logging
      log.error("Error capturing log event", e);
    }
  }

  /**
   * Checks if a log event should be captured based on configuration.
   *
   * @param event the log event
   * @param config capture configuration
   * @return true if event should be captured
   */
  private boolean shouldCapture(LogEvent event, QAPLogCaptureConfig config) {
    // Check log level
    QAPLogLevel qapLevel = convertLevel(event.getLevel());
    if (!config.shouldCapture(qapLevel)) {
      return false;
    }

    // Check logger name pattern
    String loggerName = event.getLoggerName();
    if (!config.matchesLoggerPattern(loggerName)) {
      return false;
    }

    return true;
  }

  /**
   * Converts a Log4j2 LogEvent to QAPLogEntry.
   *
   * @param event the log event
   * @param config capture configuration
   * @return QAPLogEntry
   */
  private QAPLogEntry convertLogEvent(LogEvent event, QAPLogCaptureConfig config) {
    QAPLogEntry.Builder builder = QAPLogEntry.builder();

    // Timestamp
    builder.timestamp(Instant.ofEpochMilli(event.getTimeMillis()));

    // Level
    builder.level(convertLevel(event.getLevel()));

    // Logger name
    builder.loggerName(event.getLoggerName());

    // Thread name
    builder.threadName(event.getThreadName());

    // Message
    Message message = event.getMessage();
    String messageStr = message != null ? message.getFormattedMessage() : null;
    if (messageStr != null && messageStr.length() > config.getMaxMessageLength()) {
      messageStr = messageStr.substring(0, config.getMaxMessageLength()) + "... [truncated]";
    }
    builder.message(messageStr);

    // Throwable
    Throwable throwable = event.getThrown();
    if (throwable != null && config.isCaptureStackTraces()) {
      builder.throwableMessage(throwable.toString());

      // Stack trace
      StackTraceElement[] stackTrace = throwable.getStackTrace();
      if (stackTrace != null && stackTrace.length > 0) {
        String[] stackTraceLines = new String[Math.min(stackTrace.length, 50)]; // Limit to 50 lines
        for (int i = 0; i < stackTraceLines.length; i++) {
          stackTraceLines[i] = stackTrace[i].toString();
        }
        builder.stackTrace(stackTraceLines);
      }
    }

    // MDC (ThreadContext in Log4j2)
    if (config.isIncludeMdc()) {
      Map<String, String> contextMap = event.getContextData().toMap();
      if (contextMap != null && !contextMap.isEmpty()) {
        builder.mdc(contextMap);
      }
    }

    // Markers
    if (config.isIncludeMarkers()) {
      org.apache.logging.log4j.Marker marker = event.getMarker();
      if (marker != null) {
        Set<String> markers = new HashSet<>();
        collectMarkers(marker, markers);
        if (!markers.isEmpty()) {
          builder.markers(markers);
        }
      }
    }

    return builder.build();
  }

  /**
   * Recursively collects all markers (including parents).
   *
   * @param marker the marker
   * @param result set to collect marker names
   */
  private void collectMarkers(org.apache.logging.log4j.Marker marker, Set<String> result) {
    if (marker == null) {
      return;
    }
    result.add(marker.getName());
    if (marker.hasParents()) {
      org.apache.logging.log4j.Marker[] parents = marker.getParents();
      for (org.apache.logging.log4j.Marker parent : parents) {
        collectMarkers(parent, result);
      }
    }
  }

  /**
   * Converts Log4j2 Level to QAPLogLevel.
   *
   * @param level Log4j2 level
   * @return QAPLogLevel
   */
  private QAPLogLevel convertLevel(org.apache.logging.log4j.Level level) {
    if (level == null) {
      return QAPLogLevel.INFO;
    }

    switch (level.getStandardLevel()) {
      case TRACE:
        return QAPLogLevel.TRACE;
      case DEBUG:
        return QAPLogLevel.DEBUG;
      case INFO:
        return QAPLogLevel.INFO;
      case WARN:
        return QAPLogLevel.WARN;
      case ERROR:
        return QAPLogLevel.ERROR;
      case FATAL:
        return QAPLogLevel.FATAL;
      default:
        return QAPLogLevel.INFO;
    }
  }

  /**
   * Adds a log entry to the buffer for a specific test. Safe to call from any thread.
   *
   * <p>Uses CopyOnWriteArrayList for thread-safe add operations without explicit locking.
   * The size check is intentionally racy (may slightly exceed max) to avoid synchronization
   * overhead in the hot path - this is acceptable since maxEntriesPerTest is a soft limit.
   *
   * @param testId test identifier
   * @param logEntry log entry to add
   * @param state capture state containing the buffer and config
   */
  private void addLogEntry(String testId, QAPLogEntry logEntry, CaptureState state) {
    int currentSize = state.logs.size();
    int maxEntries = state.config.getMaxEntriesPerTest();

    if (currentSize < maxEntries) {
      state.logs.add(logEntry);
    } else if (!state.maxEntriesWarningLogged) {
      // Use flag to log warning only once per test
      state.maxEntriesWarningLogged = true;
      log.warn("Max log entries ({}) reached for test: {}", maxEntries, testId);
    }
  }

  /**
   * Checks if there are any active captures.
   *
   * @return true if at least one test is actively capturing logs
   */
  public boolean hasActiveCaptures() {
    return !captureStates.isEmpty();
  }

  /** Clears all capture buffers and active captures. Should be called when the appender is stopped. */
  public void cleanupThreadLocals() {
    captureStates.clear();
    log.debug("Cleaned up capture buffers");
  }
}
```
