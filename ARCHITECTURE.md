# Migration Plugin Architecture

## Overview

The Migration plugin is a WPEFramework (Thunder) service that manages device migration status and boot type information for RDK entertainment devices transitioning between operating system environments (e.g., from legacy systems to ENTOS). It provides a standardized interface for tracking migration progress and determining boot contexts.

## System Architecture

### Component Structure

```
┌─────────────────────────────────────────────────────┐
│              Thunder Framework                       │
│  ┌───────────────────────────────────────────────┐  │
│  │         Migration Plugin (JSONRPC)            │  │
│  │  ┌─────────────────────────────────────────┐  │  │
│  │  │    Migration.cpp/.h (Plugin Shell)      │  │  │
│  │  │  - Plugin Lifecycle Management          │  │  │
│  │  │  - JSONRPC Interface Registration       │  │  │
│  │  └─────────────────────────────────────────┘  │  │
│  │                     │                          │  │
│  │                     ▼                          │  │
│  │  ┌─────────────────────────────────────────┐  │  │
│  │  │  MigrationImplementation.cpp/.h         │  │  │
│  │  │  - Business Logic                       │  │  │
│  │  │  - File I/O Operations                  │  │  │
│  │  │  - RFC Integration                      │  │  │
│  │  └─────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
                      │
         ┌────────────┼────────────┐
         ▼            ▼            ▼
    ┌─────────┐  ┌─────────┐  ┌─────────┐
    │  RFC    │  │  File   │  │  Apps   │
    │  API    │  │ System  │  │         │
    └─────────┘  └─────────┘  └─────────┘
```

### Core Components

#### 1. Migration Plugin (Migration.cpp/h)
- **Purpose**: WPEFramework plugin shell and lifecycle manager
- **Responsibilities**:
  - Plugin initialization and deinitialization
  - Connection management with plugin host
  - JSONRPC interface registration via `Exchange::JMigration`
  - Manages `MigrationImplementation` object lifecycle
- **Interfaces**: `PluginHost::IPlugin`, `PluginHost::JSONRPC`, `Exchange::IMigration`

#### 2. MigrationImplementation (MigrationImplementation.cpp/h)
- **Purpose**: Core business logic implementation
- **Responsibilities**:
  - Migration status management (get/set operations)
  - Boot type information retrieval
  - File system persistence
  - RFC (Remote Feature Control) parameter access
- **Key Methods**:
  - `GetBootTypeInfo()`: Reads boot type from `/tmp/bootType`
  - `SetMigrationStatus()`: Writes status to `/opt/secure/persistent/MigrationStatus`
  - `GetMigrationStatus()`: Reads status from RFC parameter `Device.DeviceInfo.Migration.MigrationStatus`

### Data Flow

#### Migration Status Write Operation
```
Client Request (setMigrationStatus)
    │
    ▼
JSONRPC Interface
    │
    ▼
MigrationImplementation::SetMigrationStatus()
    │
    ├─► Status enum → string mapping
    │
    ├─► File I/O: /opt/secure/persistent/MigrationStatus
    │
    └─► Return success/error
```

#### Migration Status Read Operation
```
Client Request (getMigrationStatus)
    │
    ▼
JSONRPC Interface
    │
    ▼
MigrationImplementation::GetMigrationStatus()
    │
    ├─► RFC API call (getRFCParameter)
    │
    ├─► Read TR181 parameter: Device.DeviceInfo.Migration.MigrationStatus
    │
    ├─► String → Status enum mapping
    │
    └─► Return status value
```

## Technical Implementation Details

### Migration Status States
The plugin supports eight distinct migration states:
- `NOT_STARTED`: Migration has not begun
- `NOT_NEEDED`: Device does not require migration
- `STARTED`: Migration process initiated
- `PRIORITY_SETTINGS_MIGRATED`: Critical settings transferred
- `DEVICE_SETTINGS_MIGRATED`: Device configuration migrated
- `CLOUD_SETTINGS_MIGRATED`: Cloud sync settings transferred
- `APP_DATA_MIGRATED`: Application data migrated
- `MIGRATION_COMPLETED`: All migration steps finished

### Boot Types
Four boot type scenarios are supported:
- `BOOT_INIT`: Initial boot/factory reset state
- `BOOT_NORMAL`: Standard operational boot
- `BOOT_MIGRATION`: Boot in migration mode
- `BOOT_UPDATE`: Boot for system update

### Dependencies

#### External Libraries
- **WPEFramework Core**: Plugin framework and RPC infrastructure
- **entservices-apis**: Interface definitions (`IMigration`)
- **rfcapi**: Remote Feature Control parameter access
- **CompileSettingsDebug**: Build configuration

#### Helper Utilities
- `UtilsLogging.h`: Logging macros (LOGINFO, LOGERR)
- `UtilsgetFileContent.h`: File content parsing utilities

### File System Layout
```
/opt/secure/persistent/
    └── MigrationStatus          # Persistent migration status storage

/tmp/
    └── bootType                 # Boot type configuration (BOOT_TYPE property)
```

### Error Handling
- `ERROR_NONE`: Successful operation
- `ERROR_FILE_IO`: File system access failure
- `ERROR_INVALID_PARAMETER`: Invalid migration status value
- RFC API errors mapped to `ERROR_FILE_IO`

## Build System Integration

### CMake Configuration
- Plugin built as shared library: `WPEFrameworkMigration.so`
- Implementation library: `WPEFrameworkMigrationImplementation.so`
- Installation path: `${CMAKE_INSTALL_PREFIX}/lib/${STORAGE_DIRECTORY}/plugins`
- Dependencies resolved via `find_package(${NAMESPACE}Plugins)`

### Test Framework
- **L1 Tests**: Unit tests with mocked dependencies
- **L2 Tests**: Integration tests with Thunder framework
- Mock infrastructure from `entservices-testframework`

## Performance Characteristics

### Resource Utilization
- **Memory**: Minimal footprint (~1-2 MB resident)
- **CPU**: Event-driven, negligible CPU usage when idle
- **I/O**: Lightweight file operations, no continuous polling

### Response Times
- Status read/write operations: < 100ms typical
- RFC parameter access: < 200ms typical
- Plugin initialization: < 1 second

## Security Considerations

- Migration status stored in secure persistent location (`/opt/secure/persistent/`)
- RFC parameters access controlled through platform security mechanisms
- No credential storage or sensitive data handling within plugin
- Standard WPEFramework security token validation applies

## Future Extensibility

The plugin architecture supports extension through:
- Additional migration status states
- Extended boot type scenarios
- Event notifications for status changes
- Integration with telemetry services for migration tracking
