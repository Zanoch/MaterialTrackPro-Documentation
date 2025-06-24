# Admin Panel Technical Documentation

## Overview

The MaterialTrackPro Admin Panel is a React-based administrative interface for material tracking and order management across multiple locations. It provides role-based access control with 10 distinct user roles, each with specialized interfaces and workflows.

## Technology Stack

- **Frontend Framework**: React 18.3.1 with Vite 6.0.1
- **UI Components**: shadcn/ui components built on Radix UI primitives
- **Styling**: Tailwind CSS with custom design system
- **Authentication**: AWS Amplify/Cognito for role-based access control
- **API Communication**: AWS API Gateway integration
- **Build Tool**: Vite with React SWC plugin
- **Icon Library**: Lucide React
- **Additional Libraries**:
  - React Hook Form with resolvers for form management
  - QR Scanner for barcode functionality
  - Date-fns and Luxon for date handling
  - XLSX for Excel export functionality
  - Sonner for toast notifications

## Architecture

### Application Structure

```
src/
├── components/
│   ├── auth/           # Authentication components and context
│   ├── ui/             # shadcn/ui component library
│   ├── ApprovalStatus.jsx
│   └── OrderStatus.jsx
├── pages/              # Role-specific page components
├── lib/
│   └── utils.js        # Utility functions
└── main.jsx           # Application entry point
```

### Role-Based Routing System

The application uses a dynamic routing system based on AWS Cognito groups:

1. **Authentication Flow**: Users authenticate through AWS Cognito
2. **Group Extraction**: The system extracts the user's Cognito group from the access token
3. **Dynamic Import**: Components are loaded dynamically based on the group name
4. **Fallback Handling**: Unauthorized users are redirected to an error page

```javascript
// Dynamic component loading in main.jsx
const RootComponent = lazy(() =>
  fetchAuthSession()
    .then(({ tokens }) =>
      import(`./pages/${tokens.accessToken.payload["cognito:groups"][0]
        .split("-")
        .map((word) => word.replace(/(?<=[A-Z]).+/, (match) => match.toLowerCase()))
        .join("")}.jsx`)
    )
    .catch(() => import("./pages/Unauthorized.jsx"))
);
```

## User Roles and Components

The system supports 10 distinct user roles, each mapped to specific page components:

| Role | Component | Description |
|------|-----------|-------------|
| Master Admin | `MasterAdmin.jsx` | System-wide administration and user management |
| Grandpass Admin | `GrandpassAdmin.jsx` | Location-specific administration |
| Grandpass Officer | `GrandpassOfficer.jsx` | Operational management at Grandpass |
| Grandpass Supervisor | `GrandpassSupervisor.jsx` | Supervisory functions at Grandpass |
| Grandpass Gatepass | `GrandpassGatepass.jsx` | Gate management and security |
| Grandpass Dev Machine | `GrandpassDevMachine.jsx` | Development and debugging tools |
| Seeduwa Admin | `SeeduwaAdmin.jsx` | Location-specific administration |
| Seeduwa Officer | `SeeduwaOfficer.jsx` | Operational management at Seeduwa |
| Trader Officer | `TraderOfficer.jsx` | Trader request management |
| Trader Supervisor | `TraderSupervisor.jsx` | Trader oversight and approvals |

## Key Components

### Authentication System

- **Location**: `src/components/auth/`
- **Primary Component**: `Authenticator`
- **Context**: `AuthContext` for state management
- **Features**:
  - AWS Cognito integration
  - Automatic session management
  - Role-based access control
  - Login/logout functionality

### UI Component Library

Built on shadcn/ui components with Radix UI primitives:

- **Configuration**: `components.json` defines the shadcn/ui setup
- **Location**: `src/components/ui/`
- **Key Components**:
  - Tables for data display
  - Dialogs for user interactions
  - Buttons with consistent styling
  - Form controls (inputs, selects, checkboxes)
  - Alert dialogs for confirmations
  - Cards for content organization
  - Tabs for interface organization

### Status Components

- **OrderStatus.jsx**: Displays and manages order status information
- **ApprovalStatus.jsx**: Handles approval workflow indicators

## API Integration

### Communication Pattern

The application communicates with backend services through AWS API Gateway:

```javascript
import { get, post, del } from "aws-amplify/api";

// Example API call
const { response } = get({ apiName: "MTP-API", path: "/admin/user" });
const userData = await response.then(({ body }) => body.json());
```

### Authentication Headers

- Automatic token inclusion through AWS Amplify
- Role-based authorization at the API level
- Session management and token refresh

## Development Setup

### Prerequisites

- Node.js (latest LTS version)
- npm or yarn package manager

### Installation and Development

```bash
# Install dependencies
npm install

# Start development server
npm run dev

# Build for production
npm run build

# Preview production build
npm run preview

# Run linting
npm run lint
```

### Environment Configuration

- **AWS Configuration**: `config.json` (not in repository)
- **Vite Configuration**: `vite.config.js`
- **Tailwind Configuration**: `tailwind.config.js`
- **ESLint Configuration**: `eslint.config.js`

## Design System

### Styling Approach

- **Primary**: Tailwind CSS utility classes
- **Components**: shadcn/ui design system
- **Customization**: CSS variables for theme consistency
- **Responsive**: Mobile-first responsive design

### Color Scheme

- Background: `#f1f1e9` (light cream)
- Text: `#485935` (dark green)
- Borders: `#93a267` (medium green)
- Accent colors defined through CSS variables

## Build and Deployment

### Build Configuration

- **Build Tool**: Vite with optimized production builds
- **Code Splitting**: Lazy loading for role-specific components
- **Tree Shaking**: Automatic unused code elimination
- **Asset Optimization**: Automatic image and asset optimization

### Development Features

- **Hot Module Replacement**: Instant updates during development
- **TypeScript Support**: JSDoc for type hints (no full TypeScript)
- **SSL Support**: Basic SSL plugin for HTTPS development

## Security Considerations

- **Authentication**: AWS Cognito integration with MFA support
- **Authorization**: Role-based access control at component level
- **API Security**: Authenticated API requests with automatic token management
- **Session Management**: Automatic session handling and refresh
- **Route Protection**: Dynamic route loading based on user roles

## Performance Optimizations

- **Code Splitting**: Role-based lazy loading reduces initial bundle size
- **Component Lazy Loading**: Dynamic imports for better performance
- **Optimized Dependencies**: Tree-shakable libraries and minimal bundle size
- **Caching**: Browser caching for static assets and API responses

## Troubleshooting

### Common Issues

1. **Authentication Failures**: Check AWS Cognito configuration
2. **Role Access Issues**: Verify user group assignments in Cognito
3. **API Connection Problems**: Validate API Gateway configuration
4. **Build Failures**: Check dependency versions and Node.js compatibility

### Development Tools

- **Browser DevTools**: React DevTools extension recommended
- **Network Inspection**: Monitor API calls and responses
- **Console Logging**: Comprehensive error logging throughout the application