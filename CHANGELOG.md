# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.0] - 2025-12-03

### Added
- Initial public release
- SSH-based monitoring of WAS-110 XGS-PON ONU
- MQTT Auto Discovery for Home Assistant integration
- Real-time optical metrics (RX/TX power in dBm and mW)
- Temperature monitoring (optic module, CPU zones)
- Voltage and laser bias current monitoring
- PON link status with detailed state codes
- Ethernet speed detection
- Device information sensors (vendor, part number, firmware bank)
- Bridge uptime and statistics tracking
- Test mode for validation
- Debug mode for verbose logging
- Dockerfile for containerized deployment
- Docker Compose configuration
- Example Home Assistant dashboard configuration

### Technical
- Uses native SSH via subprocess (not paramiko) for Dropbear compatibility
- Combined SSH commands to avoid device rate limiting
- Binary EEPROM parsing (base64 encoded over SSH)
- Ping-before-SSH connectivity check
- Automatic MQTT reconnection handling

## [Unreleased]

### Planned
- HACS integration packaging
- Optional SSH connection multiplexing (ControlMaster)
