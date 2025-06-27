# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a React Native joystick component library (@korsolutions/react-native-joystick) that provides touch-based joystick controls for React Native applications. The library supports Expo, React Native, and web platforms with gesture handling.

## Development Commands

- `npm run build` or `tsc --project tsconfig.build.json` - Build the TypeScript library to `lib/` directory
- `npm run prepublish` or `tsc` - Compile TypeScript using default config (used before publishing)

### Example App Commands (in example/ directory)
- `npm start` or `expo start` - Start Expo development server
- `npm run android` or `expo start --android` - Run on Android
- `npm run ios` or `expo start --ios` - Run on iOS  
- `npm run web` or `expo start --web` - Run in web browser

## Architecture

This is a library package with a separate example application:

### Main Library Structure
- **Entry Point**: `src/index.tsx` - Exports the main `ReactNativeJoystick` component and TypeScript types
- **Core Component**: `ReactNativeJoystick` - Main joystick component using react-native-gesture-handler for touch interactions
- **Types**: `src/types.ts` - TypeScript interfaces for props and events
- **Utilities**: `src/utils.ts` - Mathematical utilities for angle/distance calculations and coordinate transformations
- **Build Output**: `lib/` directory (generated) - Contains compiled JavaScript and TypeScript declarations

### Example Application
- **Location**: `example/` directory - Expo-based demo app
- **Purpose**: Demonstrates library usage and serves as development testing environment
- **Dependencies**: Uses file reference to main library (`"file:../"`)

## Key Technical Details

### Gesture Handling
- Uses `react-native-gesture-handler` for cross-platform touch interactions
- Implements pan gestures with start, move, and end callbacks
- Platform-specific coordinate handling (web vs native)

### Mathematical Calculations
- Real-time angle and distance calculations from center point
- Force calculation based on distance from center
- Coordinate clamping to keep joystick within circular bounds
- Conversion between degrees and radians

### Cross-Platform Considerations
- Web platform has reversed Y-axis handling (`Platform.OS === 'web'`)
- Transform styles include `rotateX: "180deg"` for proper visual orientation
- Uses hex color codes with opacity for styling (`${color}55`, `${color}bb`)

## Build System

- **TypeScript**: Strict mode enabled with ESNext target
- **Build Configs**: Separate configs for development (`tsconfig.json`) and library build (`tsconfig.build.json`)
- **Output**: Compiles to CommonJS in `lib/` directory with declaration files
- **Peer Dependencies**: React, React Native, and react-native-gesture-handler must be provided by consuming app

## Testing and Development

Use the example app for testing changes - it provides a live development environment with the library linked as a local file dependency. The example app supports all platforms (iOS, Android, web) for comprehensive testing.