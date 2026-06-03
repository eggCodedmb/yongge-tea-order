# Project Overview: dian-wu (点物)

A modern mobile ordering application for "Yi He Tang" (益禾堂) built with **Uni-app x (UTS)** and **Vue 3**. This project targets multiple platforms including mobile apps (Android/iOS) and mini-programs (Weixin, Alipay, etc.).

## 🛠 Main Technologies
- **Framework:** Uni-app x (UTS) + Vue 3
- **State Management:** Pinia
- **UI Components:** uni-ui
- **Language:** UTS (Uni-app TypeScript)
- **Styling:** SCSS / CSS

## 📁 Architecture and Directory Structure
- `pages/`: Application pages (Index, Menu, Detail, Order, Profile, Checkout).
- `store/`: Global state management using Pinia.
    - `app.uts`: Application-level state.
    - `cart.uts`: Shopping cart logic and state.
    - `user.uts`: User authentication and profile state.
- `utils/`: Utility functions.
    - `request.uts`: Centralized API request handling with base URL configuration and token injection.
- `static/`: Static assets such as banners, logos, and tab bar icons.
- `main.uts`: The entry point for the application.
- `App.uvue`: The root component.
- `pages.json`: Configuration for routing, page styles, and the bottom tab bar.
- `manifest.json`: Cross-platform configuration and app identity.

## 🚀 Building and Running

### Development
Currently, the project is configured for the Uni-app ecosystem. 
- **Tooling:** Highly recommended to use **HBuilderX** for the best development experience with UTS.
- **Running:** Use the "Run" menu in HBuilderX to deploy to a browser, emulator, or real device.
- **Dependencies:** 
  ```bash
  npm install
  ```

### Key Commands (TODO)
- [ ] Document specific build commands for different platforms if using CLI.
- [ ] Add testing scripts to `package.json`.

## 📜 Development Conventions

### State Management
- All global state must be managed through Pinia stores in the `store/` directory.
- Use descriptive actions and getters to keep component logic clean.

### API Requests
- Always use the `request` function from `utils/request.uts` for network calls.
- Define interfaces for API responses to ensure type safety.
- The base URL is currently set to `http://192.168.2.240:8800`.

### Typing (UTS)
- Embrace the strong typing features of UTS. Define interfaces for data structures (e.g., `CartItem`, `ApiResponse`).
- Avoid using `any` where possible to maintain the benefits of the type system.

### UI & Styling
- Leverage `uni-ui` components for consistent look and feel.
- Global styles and theme colors are defined in `uni.scss` and `pages.json`.
