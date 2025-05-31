# Frontend Architecture: Mobile-First 3D Creation App 

## 1. High-Level Summary

The frontend is a **mobile-first, cross-platform application** built using **React Native and Expo**. Its primary purpose for the **MVP (Minimum Viable Product)** is to allow users to **generate 3D models from various inputs (text, image, sketch, photo)** via a Backend-for-Frontend (BFF) that interacts with Tripo AI and OpenAI.

**Key Architectural Principles (aligned with BFF API v1.1.0):**
*   **Client-Generated `task_id`**: For each new generation job/workspace item, the client generates a unique `task_id`.
*   **Client Manages Supabase Records**: The client is responsible for creating and managing records in its own Supabase tables (`input_assets`, `concept_images`, `models`) to track the overall job and its associated assets.
*   **Inputs via Supabase**: For image/sketch inputs, the client uploads the asset to its Supabase Storage first, creates a record in `input_assets` with the asset URL, and then provides this URL to the BFF.
*   **Polling & Status**: The client polls the BFF's `/tasks/{task_id}/status?service={service}` endpoint for real-time AI step status. Upon completion of an AI step by the BFF (which includes storing the asset in Supabase Storage and updating the respective metadata table like `concept_images` or `models`), the client retrieves the final asset URL. The client primarily uses its own Supabase tables (`input_assets`, `concept_images`, `models`) as the source of truth for persisted asset locations and statuses.

The app handles user authentication (e.g., Supabase Auth) and is designed for intuitive mobile interaction.

## 2. Features

*   **User Authentication**: Sign-up, sign-in, profile management (using Supabase Auth). Supports email/password, and potentially OAuth providers (e.g., Google, Apple).
*   **Supabase Integration**: Client directly interacts with its Supabase instance for:
    *   Uploading input images to Supabase Storage.
    *   Creating and managing records in `input_assets`, `concept_images`, and `models` tables (tracking `task_id`, prompts, styles, asset URLs, statuses).
*   **Home Screen**:
    *   **Main Navigation Grid (2x2)**: 
        *   Text to 3D
        *   Image/Photo to 3D
        *   Sculpt Model
        *   Color Model
    *   **Sculpt Model Modal**: (Accessed via "Sculpt Model" button) Floating screen with options to load existing model (from workspace/public) or start from scratch (basic sphere).
    *   **Color Model Modal**: (Accessed via "Color Model" button) Floating screen with options to load existing model without texture for coloring workflow.
    *   **Workspace Section**: Mini previews prioritizing pending/processing models with status indicators, delete options, and "View All" button.
    *   **Public Models Section**: Horizontal scroll of available stock models and community shared models (post-MVP).
*   **Text to 3D Screen**:
    *   User inputs text prompt and selects style.
    *   Client generates `task_id`, creates record in `input_assets` (type: 'text_prompt').
    *   **Advanced Options**: Generate concept toggle.
    *   **Style Options**: None, Ghibli, Simpson, South Park, Anime, Cartoon, Claymation, Muppet, Metal, Wood, Glass.
    *   Control panel on the left, preview area on the right (tablet/desktop).
*   **Image/Photo to 3D Screen**:
    *   Combined photo/image input with a toggle, camera/gallery picker.
    *   Includes multi-view option for enhanced 3D reconstruction (4 photos: front, left, back, right).
    *   Client uploads to Supabase Storage, creates `task_id` and record in `input_assets` with the asset's Supabase URL.
    *   **Advanced Options**: Multi-view capture, Generate concept toggle.
    *   **Style Options**: None, Ghibli, Simpson, South Park, Anime, Cartoon, Claymation, Muppet, Metal, Wood, Glass.
    *   Control panel on the left, preview area on the right (tablet/desktop).
*   **Workspace Screen**: 
    *   Full view of all user models with infinite scrolling.
    *   Status indicators (pending, processing, complete).
    *   **Smart Model Navigation**: Clicking a model takes user to:
        *   Pending/Processing jobs → relevant last step (Text to 3D, Image/Photo to 3D, or Concept Selection).
        *   Complete jobs → 3D Sculpting Viewer.
*   **Concept Selection Screen**:
    *   If concept generation was triggered, displays concepts for the active `task_id` (fetched from client's `concept_images` table).
    *   User selects a concept to proceed to 3D model generation.
*   **3D Sculpting Viewer**: Renders `.glb` models using Supabase URLs (from `models` table) and integrates with existing sculpting tools.
*   **Loading/Processing Indicators**: Real-time status updates for BFF calls (polling `/tasks/{task_id}/status`) and client-side Supabase operations.

## 3. Architecture & Components

*   **Framework**: React Native + Expo.
*   **Language**: TypeScript for new features, JavaScript for existing sculpting tools.
*   **UI Components**: Existing UI components, enhanced with new TypeScript components.
*   **3D Rendering**: `react-three-fiber`, `@react-three/drei`, `expo-gl`.
*   **State Management**: **Redux Toolkit** (extending existing store with new TypeScript slices for API features).
*   **Authentication**: Supabase Auth (via `@supabase/supabase-js` library). 
    *   Project: MakeIt3D (configured via environment variables)
    *   Anon Key: Configured via environment variables for JWT validation
    *   Utilizes AsyncStorage or SecureStore for session persistence
    *   JWT tokens sent to BFF via Authorization header
*   **Data Fetching & Client-Server State**: Redux Toolkit Query (RTK Query) for API calls, integrated with existing Redux store.
*   **Supabase Client Library**: For direct database and storage interaction.

### API Communication (BFF Centric - Refactored):
    *   **Authentication**: Client includes JWT token in `Authorization: Bearer <token>` header for all BFF requests.
    *   Client generates `task_id` for each job.
    *   For image inputs: Client uploads to its Supabase Storage, creates a record in its `input_assets` table, then calls BFF with the `task_id` and the asset's Supabase URL.
    *   Calls BFF endpoints (e.g., `/generate/image-to-image`, `/generate/text-to-model`) with `task_id` and relevant parameters (including Supabase URLs for inputs).
    *   **User Isolation**: BFF validates JWT token and extracts `user_id` to ensure data isolation via RLS policies.
    *   Polls BFF `GET /tasks/{task_id}/status?service={service}` for real-time AI step status.
    *   BFF, upon completing an AI step, stores the output asset in Supabase Storage and updates the client's `concept_images` or `models` table (including status and final asset URL).
    *   Client primarily queries its own Supabase tables (`input_assets`, `concept_images`, `models`) to display persisted job status and asset URLs.

### Sketching Canvas (for Sketch-to-Image)
    * `react-native-skia`.

### Image Inpainting (for Image-Inpaint)

*   **Canvas/Mask Editor**: Allows users to draw masks on uploaded images to specify areas for inpainting
*   **Image Upload**: Users upload a base image for inpainting
*   **Mask Creation**: Users can draw or upload a mask to specify which areas to modify
*   **Prompt Input**: Text description of what should be painted in the masked areas
*   **Provider Selection**: Currently supports Recraft only
*   **Style Options**: Realistic image, digital art, etc. (Recraft-specific styles)
*   Calls BFF endpoint `/generate/image-inpaint` with `task_id` and relevant parameters (including Supabase URLs for image and mask, plus text prompt).

## 4.5. Complete UI Wireframes & Design Specifications

### Device Layout Strategy

**Responsive Design Approach:**
- **Tablets/Desktop (Landscape)**: Control panel on left, main preview on right (for creation screens).
- **Phones (Landscape)**: Similar to tablet layout with smaller/collapsible control panel (for creation screens).
- **Phones (Portrait)**: Stacked vertically - control panel on top, preview below (for creation screens).

---

## 1. Home Screen Wireframes

### Tablet/Desktop Layout (Landscape)
```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              MakeIt3D Home                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│ ┌──────────────────────────┐  ┌──────────────────────────┐                │
│ │                          │  │                          │                │
│ │       Text to 3D         │  │    Image/Photo to 3D     │                │
│ │                          │  │  (Toggle: Image/Photo)   │                │
│ └──────────────────────────┘  └──────────────────────────┘                │
│                                                                             │
│ ┌──────────────────────────┐  ┌──────────────────────────┐                │
│ │                          │  │                          │                │
│ │      Sculpt Model        │  │       Color Model        │                │
│ │                          │  │                          │                │
│ └──────────────────────────┘  └──────────────────────────┘                │
├─────────────────────────────────────────────────────────────────────────────┤
│                              Your Workspace                                │
│ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ [View All] │
│ │ 🔄  │ │ ⚡  │ │ ✓   │ │ ✓   │ │ ✓   │ │ ✓   │ │ ✓   │ │ ✓   │           │
│ │Model│ │Model│ │Model│ │Model│ │Model│ │Model│ │Model│ │Model│           │
│ └─────┘ └─────┘ └─────┘ └─────┘ └─────┘ └─────┘ └─────┘ └─────┘           │
├─────────────────────────────────────────────────────────────────────────────┤
│                             Public Models                                  │
│ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ [Explore] │
│ │Stock│ │Stock│ │Stock│ │Stock│ │Stock│ │Stock│ │Stock│ │Stock│           │
│ │Model│ │Model│ │Model│ │Model│ │Model│ │Model│ │Model│ │Model│           │
│ └─────┘ └─────┘ └─────┘ └─────┘ └─────┘ └─────┘ └─────┘ └─────┘           │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Phone Layout (Portrait)
```
┌─────────────────────────────┐
│        MakeIt3D Home        │
├─────────────────────────────┤
│ ┌─────────────────────────┐ │
│ │       Text to 3D        │ │
│ └─────────────────────────┘ │
│ ┌─────────────────────────┐ │
│ │   Image/Photo to 3D     │ │
│ └─────────────────────────┘ │
│ ┌─────────────────────────┐ │
│ │      Sculpt Model       │ │
│ └─────────────────────────┘ │
│ ┌─────────────────────────┐ │
│ │       Color Model       │ │
│ └─────────────────────────┘ │
├─────────────────────────────┤
│      Your Workspace        │
│ ┌───┐ ┌───┐ ┌───┐ [View All]│
│ │🔄 │ │⚡ │ │✓  │          │
│ └───┘ └───┘ └───┘          │
│ ┌───┐ ┌───┐ ┌───┐          │
│ │✓  │ │✓  │ │✓  │          │
│ └───┘ └───┘ └───┘          │
├─────────────────────────────┤
│      Public Models         │
│ ┌───┐ ┌───┐ ┌───┐ [Explore]│
│ │📦 │ │📦 │ │📦 │          │
│ └───┘ └───┘ └───┘          │
└─────────────────────────────┘
```

---

## 2. Text to 3D Screen Wireframes

### Tablet/Desktop Layout (Landscape)
```
┌───────────────────────────────────────────────────────────────────────────────────┐
│                                Text to 3D Creation                                 │
├─────────────────────────┬───────────────────────────────────────────────────────────┤
│ CONTROL PANEL           │                    PREVIEW AREA                          │
│ (Text to 3D)            │                                                           │
│                         │ ┌───────────────────────────────────────────────────────┐ │
│ Text Prompt:            │ │                                                       │ │
│ ┌─────────────────────┐ │ │              Text Prompt Preview                      │ │
│ │ Enter prompt here   │ │ │                                                       │ │
│ │ ...                 │ │ │         "A cute cartoon dragon"                       │ │
│ └─────────────────────┘ │ │                                                       │ │
│                         │ │      [Concept Preview - if generated]                 │ │
│ ☐ Generate concept      │ │                                                       │ │
│                         │ └───────────────────────────────────────────────────────┘ │
│ Style:                  │                                                           │
│ ┌──────┐ ┌──────┐     │                                                           │
│ │  🎨  │ │  🎭  │ ... │                                                           │
│ └──────┘ └──────┘     │                                                           │
│                         │                                                           │
│ ┌─────────────────────┐ │                                                           │
│ │   Generate (10c)    │ │                                                           │
│ └─────────────────────┘ │                                                           │
└─────────────────────────┴───────────────────────────────────────────────────────────┘
```

### Phone Layout (Portrait) - Text to 3D
```
┌─────────────────────────────┐
│     Text to 3D Creation     │
├─────────────────────────────┤
│        CONTROL PANEL        │
│ Text Prompt:                │
│ ┌─────────────────────────┐ │
│ │   Enter prompt here...  │ │
│ └─────────────────────────┘ │
│ ☐ Generate concept          │
│ Style: ┌───┐ ┌───┐ ...    │
│        │🎨 │ │🎭 │        │
│        └───┘ └───┘        │
│ ┌─────────────────────────┐ │
│ │     Generate 10c        │ │
│ └─────────────────────────┘ │
├─────────────────────────────┤
│       PREVIEW AREA          │
│ ┌─────────────────────────┐ │
│ │  "A cute cartoon dragon"│ │
│ │                         │ │
│ │ [Concept Preview]       │ │
│ └─────────────────────────┘ │
└─────────────────────────────┘
```

---

## 3. Image/Photo to 3D Screen Wireframes

### Tablet/Desktop Layout (Landscape) - Image/Photo to 3D (Multi-View Example)
```
┌───────────────────────────────────────────────────────────────────────────────────┐
│                           Image/Photo to 3D Creation                             │
├─────────────────────────┬───────────────────────────────────────────────────────────┤
│ CONTROL PANEL           │                    PREVIEW AREA                          │
│ (Image/Photo)           │                                                           │
│                         │                                                           │
│ ┌─────────────────────┐ │ ┌───────────────────────────────────────────────────────┐ │
│ │    Image to 3D      │ │ │                                                       │ │
│ └─────────────────────┘ │ │           Front Photo/Image Preview                   │ │
│ ┌─────────────────────┐ │ │                                                       │ │
│ │    Photo to 3D      │ │ │         [Main Preview Area - Shows selected           │ │
│ │    (selected)       │ │ │          front image or concept output]               │ │
│ └─────────────────────┘ │ │                                                       │ │
│                         │ └───────────────────────────────────────────────────────┘ │
│ Upload Front:           │                                                           │
│ ┌─────────────────────┐ │                                                           │
│ │ [Take/Upload Photo] │ │                                                           │
│ └─────────────────────┘ │                                                           │
│                         │                                                           │
│ ☑ Multiview             │                                                           │
│   If checked:           │                                                           │
│   Left View:            │                                                           │
│   ┌───────────────────┐ │                                                           │
│   │ [Take/Upload Left]│ │                                                           │
│   └───────────────────┘ │                                                           │
│   Back View:            │                                                           │
│   ┌───────────────────┐ │                                                           │
│   │[Take/Upload Back] │ │                                                           │
│   └───────────────────┘ │                                                           │
│   Right View:           │                                                           │
│   ┌───────────────────┐ │                                                           │
│   │[Take/Upload Right]│ │                                                           │
│   └───────────────────┘ │                                                           │
│                         │                                                           │
│ ☐ Generate concept      │                                                           │
│                         │                                                           │
│ Style:                  │                                                           │
│ ┌──────┐ ┌──────┐     │                                                           │
│ │  🎨  │ │  🎭  │ ... │                                                           │
│ └──────┘ └──────┘     │                                                           │
│                         │                                                           │
│ ┌─────────────────────┐ │                                                           │
│ │  Generate (15c)     │ │                                                           │
│ └─────────────────────┘ │                                                           │
└─────────────────────────┴───────────────────────────────────────────────────────────┘
```

### Phone Layout (Portrait) - Image/Photo to 3D (Multi-View Example)
```
┌─────────────────────────────┐
│  Image/Photo to 3D Creation │
├─────────────────────────────┤
│        CONTROL PANEL        │
│ ┌─────────────────────────┐ │
│ │      Image to 3D        │ │
│ └─────────────────────────┘ │
│ ┌─────────────────────────┐ │
│ │      Photo to 3D        │ │
│ │      (selected)         │ │
│ └─────────────────────────┘ │
│                             │
│ Upload Front: [Take/Upload] │
│                             │
│ ☑ Multiview                 │
│   If checked:               │
│   Left: [T/U] Back: [T/U]   │
│   Right: [T/U]              │
│                             │
│ ☐ Generate concept          │
│ Style: ┌───┐ ┌───┐ ...    │
│        │🎨 │ │🎭 │        │
│        └───┘ └───┘        │
│ ┌─────────────────────────┐ │
│ │     Generate 15c        │ │
│ └─────────────────────────┘ │
├─────────────────────────────┤
│       PREVIEW AREA          │
│ ┌─────────────────────────┐ │
│ │                         │ │
│ │  Front Photo/Image or   │ │
│ │   Concept Preview       │ │
│ │                         │ │
│ └─────────────────────────┘ │
└─────────────────────────────┘
```

---

## 4. Concept Selection Screen Wireframes

### Tablet/Desktop Layout (Landscape)
```
┌───────────────────────────────────────────────────────────────────────────────────┐
│                                Concept Selection                                   │
├─────────────────────────┬───────────────────────────────────────────────────────────┤
│ CONTROL PANEL           │                    PREVIEW AREA                          │
│                         │                                                           │
│ Original Input:         │ ┌───────────────────────────────────────────────────────┐ │
│ ┌─────────────────────┐ │ │                                                       │ │
│ │ [Preview of       ] │ │ │             Selected Concept (Large)                  │ │
│ │ [original image/   ] │ │ │                                                       │ │
│ │ [text prompt here ] │ │ │           [Image Preview of Concept 1]                │ │
│ └─────────────────────┘ │ │                                                       │ │
│                         │ │                                                       │ │
│ Instructions:           │ └───────────────────────────────────────────────────────┘ │
│ Select one of the       │                                                           │
│ generated concepts      │ ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌───────────┐ │
│ below to proceed.       │ │ Concept 1 │ │ Concept 2 │ │ Concept 3 │ │ Concept 4 │ │
│                         │ │ (Selected)│ │           │ │           │ │ (Optional)│ │
│ Min 1, Max 4 concepts.  │ │ [Thumb 1] │ │ [Thumb 2] │ │ [Thumb 3] │ │ [Thumb 4] │ │
│                         │ └───────────┘ └───────────┘ └───────────┘ └───────────┘ │
│                         │                                                           │
│ ┌─────────────────────┐ │                                                           │
│ │Generate 3D Model    │ │                                                           │
│ │(10c)                │ │                                                           │
│ └─────────────────────┘ │                                                           │
│                         │                                                           │
│ ┌─────────────────────┐ │                                                           │
│ │Back to Generation   │ │                                                           │
│ └─────────────────────┘ │                                                           │
└─────────────────────────┴───────────────────────────────────────────────────────────┘
```

### Phone Layout (Portrait)
```
┌─────────────────────────────┐
│      Concept Selection      │
├─────────────────────────────┤
│        PREVIEW AREA         │
│ ┌─────────────────────────┐ │
│ │                         │ │
│ │    Selected Concept     │ │
│ │   [Large Img Preview]   │ │
│ │                         │ │
│ └─────────────────────────┘ │
│                             │
│ Generated Concepts (1-4):   │
│ ┌────┐ ┌────┐ ┌────┐ ┌────┐ │
│ │ C1 │ │ C2 │ │ C3 │ │ C4 │ │
│ │(sel)│      │      │ (opt)│ │
│ └────┘ └────┘ └────┘ └────┘ │
├─────────────────────────────┤
│        CONTROL PANEL        │
│ Original Input: [Truncated] │
│                             │
│ ┌─────────────────────────┐ │
│ │   Generate 3D Model     │ │
│ │         (10c)           │ │
│ └─────────────────────────┘ │
│ ┌─────────────────────────┐ │
│ │   Back to Generation    │ │
│ └─────────────────────────┘ │
└─────────────────────────────┘
```

---

## 5. Workspace Screen Wireframes

### Tablet/Desktop Layout (Landscape)
```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              Your Workspace                                │
├─────────────────┬───────────────────────────────────────────────────────────┤
│ CONTROL PANEL   │                    PREVIEW AREA                          │
│ (Filters etc.)  │                                                           │
│ Filter & Sort:  │ ┌───────────────────────────────────────────────────────┐ │
│ ┌─────────────┐ │ │                                                       │ │
│ │ All Models  │ │ │              Selected Model                           │ │
│ └─────────────┘ │ │                                                       │ │
│ ... (Filters)   │ │              [3D Preview]                             │ │
│                 │ │                                                       │ │
│ Search:         │ └───────────────────────────────────────────────────────┘ │
│ ┌─────────────┐ │                                                           │
│ │ Search...   │ │ Model Details & Actions                                   │
│ └─────────────┘ │ ...                                                       │
│                 │                                                           │
│ Your Models     │                                                           │
│ (Scrollable List)│                                                          │
│ ┌─────────────┐ │                                                           │
│ │ Model 1     │ │                                                           │
│ │ ✓ Complete  │ │                                                           │
│ │ (selected)  │ │                                                           │
│ └─────────────┘ │                                                           │
│ ...             │                                                           │
└─────────────────┴───────────────────────────────────────────────────────────┘
```

### Phone Layout (Portrait)
```
┌─────────────────────────────┐
│       Your Workspace        │
├─────────────────────────────┤
│        CONTROL PANEL        │
│ Filter: [All ▼] Sort: [▼]   │
│ ┌─────────────────────────┐ │
│ │      Search...          │ │
│ └─────────────────────────┘ │
│ Models List (Scrollable):   │
│ ┌─────────────────────────┐ │
│ │ Model 1 - ✓ Complete    │ │
│ │ (selected)              │ │
│ └─────────────────────────┘ │
│ ...                         │
│ Actions: [Sculpt] [Delete]  │
├─────────────────────────────┤
│       PREVIEW AREA          │
│ ┌─────────────────────────┐ │
│ │                         │ │
│ │   Selected Model        │ │
│ │   [3D Preview]          │ │
│ └─────────────────────────┘ │
│ Details...                  │
└─────────────────────────────┘
```

---

## 6. Modal Wireframes

### Sculpt Model Modal (Tablet/Desktop)
```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              Sculpt Model                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                       (Accessed from Home Screen)                         │
│                                                                             │
│ ┌─────────────────────┐ ┌─────────────────────┐ ┌─────────────────────┐     │
│ │                     │ │                     │ │                     │     │
│ │  Load from          │ │  Load from          │ │  Start from         │     │
│ │  Workspace          │ │  Public             │ │  Scratch            │     │
│ │                     │ │                     │ │                     │     │
│ │  [Your Models]      │ │  [Stock Models]     │ │  [Basic Sphere]     │     │
│ │                     │ │                     │ │                     │     │
│ └─────────────────────┘ └─────────────────────┘ └─────────────────────┘     │
│                                                                             │
│                              [Cancel]                                      │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Color Model Modal (Phone Portrait)
```
┌─────────────────────────────┐
│        Color Model          │
├─────────────────────────────┤
│ (Accessed from Home Screen) │
│                             │
│ ┌─────────────────────────┐ │
│ │                         │ │
│ │    Load from            │ │
│ │    Workspace            │ │
│ │                         │ │
│ │   [Your Models]         │ │
│ │   (No Texture)          │ │
│ │                         │ │
│ └─────────────────────────┘ │
│                             │
│ ┌─────────────────────────┐ │
│ │                         │ │
│ │    Load from            │ │
│ │    Public               │ │
│ │                         │ │
│ │   [Stock Models]        │ │
│ │   (No Texture)          │ │
│ │                         │ │
│ └─────────────────────────┘ │
│                             │
│        [Cancel]             │
└─────────────────────────────┘
```

---

## Design Specifications Summary

### Control Panel Behavior (for Text to 3D & Image/Photo to 3D Screens):
- **Tablets/Desktop**: Fixed left panel (e.g., occupying approx. 30-35% of screen width, or a comfortable fixed width like 350-450px, allowing for spacious internal layout), main content/preview area responsive.
- **Phones Landscape**: Collapsible left panel (approx. 250px width), swipe to show/hide.
- **Phones Portrait**: Stacked layout, control panel scrollable at top, preview area below.

### Responsive Breakpoints:
- **Mobile Portrait**: < 768px width, stacked layout.
- **Mobile Landscape**: 768px - 1024px width, side-by-side with smaller panel.
- **Tablet/Desktop**: > 1024px width, full side-by-side layout.

### Preview Area Content (Mode-Specific Screens):
- **Text to 3D Screen**: Shows text prompt and concept preview (if generated).
- **Image/Photo to 3D Screen**: Shows uploaded photos (main and multi-view) and concept preview (if generated).
- **Concept Selection Screen**: Shows large concept preview.
- **Workspace Screen**: Shows 3D model preview and details.
- **Processing States**: Shows progress indicators and status within the preview area or relevant UI element.

### Multi-View Interface Behavior (Image/Photo to 3D Screen)

**Default State (Multi-view unchecked):**
- Only main photo/image area visible.
- Toggle buttons for "Image to 3D" / "Photo to 3D" at the top of the control panel.
- Single photo/image capture/upload functionality in the main preview area.

**Multi-View Enabled State (Multi-view checked):**
- Main photo/image area remains: "Click to Take/Upload Front Photo/Image".
- Three additional photo areas appear below (or in a dedicated section of the control panel for phones) for:
  - Left view (square thumbnail area/upload button)
  - Back View (square thumbnail area/upload button) 
  - Right View (square thumbnail area/upload button)
- Each area allows independent photo capture/upload.
- All 4 photos/images required before generation can proceed.
- Generate button shows cost and remains disabled until all photos captured.

**Photo Capture States:**
- Empty state: Shows placeholder text and camera/upload icon.
- Captured/Uploaded state: Shows thumbnail preview with retake/replace option.
- Loading state: Shows upload progress indicator.

### Style Selector Layout (Control Panel)

**Style Grid (e.g., 3x2 + expandable):**
- Visible style options in a grid (e.g., 3 columns).
- Each style shows icon/preview and name.
- "..." or "More Styles" button to expand/scroll and show all available styles.
- Selected style highlighted.
- Styles: None, Ghibli, Simpson, South Park, Anime, Cartoon, Claymation, Muppet, Metal, Wood, Glass.

### Generation Options (Control Panel)

**Checkboxes:**
- ☑/☐ Multiview: Toggles 4-photo capture interface (specific to Image/Photo to 3D).
- ☑/☐ Generate concept: Enables concept generation step before 3D model (available for Text to 3D and Image/Photo to 3D).

**Generate Button:**
- Shows cost (e.g., "10c" for credits).
- Disabled state when required inputs missing.
- Loading state during generation process.
- Success state transitions to the next screen (e.g., Concept Selection or Sculpting Viewer).

## 4. Current Directory Structure & Integration Plan

**Current Structure (JavaScript-based sculpting tools with Redux):**
```
/makeit3d-frontend
|-- App.js                     # Main app entry point (existing)
|-- package.json               # Dependencies (existing, includes Redux Toolkit)
|-- babel.config.js            # Babel config (existing)
|-- /assets                    # Static assets (existing)
|-- /src
|   |-- /components            # Reusable UI components (existing)
|   |   |-- /editor            # Sculpting editor components (existing JS)
|   |   |   |-- BrushVisual.js
|   |   |   |-- Model3D.js
|   |   |   |-- ModelViewer.js
|   |   |-- /ui                # Generic UI elements (existing)
|   |-- /config                # App configuration (existing)
|   |   |-- theme.js           # UI theme (existing)
|   |-- /features              # Feature modules (existing)
|   |   |-- /advanced-editor   # Advanced sculpting features (existing JS)
|   |   |   |-- /screens
|   |   |   |   |-- AdvancedEditorScreen.js
|   |   |   |-- /components
|   |-- /hooks                 # Custom React hooks (existing)
|   |-- /navigation            # Navigation setup (existing)
|   |   |-- AppNavigator.js
|   |-- /screens               # Top-level screens (existing)
|   |   |-- ModelPickerScreen.js  # Recently refactored with Redux integration
|   |-- /state                 # Redux state management (existing)
|   |   |-- store.js           # Redux store configuration (existing)
|   |   |-- slices/            # Redux slices (existing)
|   |   |   |-- appSlice.js    # Global app state (existing)
|   |   |   |-- editorSlice.js # Sculpting tools state (existing)
|   |   |   |-- modelSlice.js  # Current model state (existing)
|   |-- /utils                 # Utility functions (existing)
|   |   |-- UndoRedoManager.js # Recently improved (existing)
```

**New API Integration Structure (TypeScript additions extending Redux):**
```
/makeit3d-frontend
|-- /src
|   |-- /api                   # NEW: API service definitions (TypeScript)
|   |   |-- bffClient.ts       # Axios/fetch instance for BFF
|   |   |-- supabaseClient.ts  # Supabase JS client instance
|   |   |-- bffApi.ts          # RTK Query API slice for BFF endpoints
|   |   |-- supabaseApi.ts     # RTK Query API slice for Supabase operations
|   |-- /components
|   |   |-- /modals            # NEW: Modal components (TypeScript)
|   |   |   |-- SculptModelModal.tsx
|   |   |   |-- ColorModelModal.tsx
|   |   |-- /generation        # NEW: Model generation components (TypeScript)
|   |   |   |-- StyleSelector.tsx
|   |   |   |-- ConceptCard.tsx
|   |   |   |-- PhotoCapture.tsx        # Single photo capture component
|   |   |   |-- MultiViewCapture.tsx    # Multi-view photo capture interface
|   |   |   |-- GenerationOptions.tsx   # Multiview & concept checkboxes
|   |   |-- /shared            # NEW: Enhanced reusable components (TypeScript)
|   |   |   |-- ModelPicker.tsx # Extracted from existing ModelPickerScreen
|   |   |   |-- ModelCard.tsx   # Reusable model display component
|   |-- /features
|   |   |-- /model-generation  # NEW: Model generation feature (TypeScript)
|   |   |   |-- /components
|   |   |   |-- /hooks         # useAppSelector, useAppDispatch wrappers
|   |   |   |-- /screens
|   |   |   |   |-- HomeScreen.tsx
|   |   |   |   |-- TextToModelScreen.tsx    # NEW
|   |   |   |   |-- ImageToModelScreen.tsx   # NEW (handles image & photo)
|   |   |   |   |-- ConceptSelectionScreen.tsx
|   |   |   |   |-- WorkspaceScreen.tsx
|   |-- /hooks
|   |   |-- useSupabaseAuth.ts # NEW: Supabase auth hook
|   |   |-- useTypedSelector.ts # NEW: Typed Redux hooks
|   |-- /state                 # EXTENDED: Redux state management
|   |   |-- store.ts           # UPDATED: TypeScript store configuration
|   |   |-- rootReducer.ts     # UPDATED: Combined reducers with types
|   |   |-- slices/            # EXTENDED: Redux slices
|   |   |   |-- appSlice.js    # EXISTING: Global app state
|   |   |   |-- editorSlice.js # EXISTING: Sculpting tools state
|   |   |   |-- modelSlice.js  # EXISTING: Current model state
|   |   |   |-- generationSlice.ts # NEW: Model generation state
|   |   |   |-- workspaceSlice.ts  # NEW: Workspace management state
|   |   |   |-- authSlice.ts   # NEW: Authentication state
|   |-- /types                 # NEW: TypeScript types
|   |   |-- api.ts             # BFF API types
|   |   |-- supabase.ts        # Supabase table types
|   |   |-- redux.ts           # Redux state types
|   |   |-- domain.ts          # Core domain types
|-- tsconfig.json              # NEW: TypeScript configuration
|-- .env                       # NEW: Environment variables
```

**Integration Strategy:**
- **Extend existing Redux store** - add new TypeScript slices alongside existing JavaScript slices
- **Reuse existing patterns** - leverage recent Redux refactoring in `ModelPickerScreen` for new components
- **Extract reusable components** - create `ModelPicker.tsx` from existing `ModelPickerScreen.js` logic
- **Keep existing sculpting tools in JavaScript** - no changes to `/features/advanced-editor/`
- **Add new TypeScript features alongside** - new `/features/model-generation/` using Redux patterns
- **Extend existing navigation** - update `AppNavigator.js` to include new screens (`TextToModelScreen`, `ImageToModelScreen`)
- **Shared state management** - unified Redux store for both existing and new features
- **Bridge existing and new** - TypeScript wrappers for existing Redux slices where needed

## 5. User & Sequence Flows (Refactored)

**Home Screen Flow**:
1.  **User Action**: Opens app, sees Home Screen with a 2x2 grid: "Text to 3D", "Image/Photo to 3D", "Sculpt Model", "Color Model".
2.  **User Action**: Can either:
    *   Click "Text to 3D" → Navigate to Text to 3D Screen.
    *   Click "Image/Photo to 3D" → Navigate to Image/Photo to 3D Screen.
    *   Click "Sculpt Model" → Open Sculpt Model Modal with three options:
        *   "Load from Workspace" → Model picker from user's completed models.
        *   "Load from Public" → Model picker from stock/community models.
        *   "Start from Scratch" → Load basic sphere into Sculpting Viewer.
        *   Selected model loads into Sculpting Mode of the sculpting tool.
    *   Click "Color Model" → Open Color Model Modal with two options:
        *   "Load from Workspace" → Model picker from user's completed models (filter for models without texture).
        *   "Load from Public" → Model picker from stock/community models (filter for models without texture).
        *   Selected model loads into Coloring Mode of sculpting tool.
    *   Click on a model in Workspace section → Smart navigation based on status.
    *   Long press or swipe on pending/processing model → Delete option (removes from Supabase).
    *   Click "View All" in Workspace section → Navigate to full Workspace Screen.
    *   Click on public model → Load directly into relevant viewer/editor.

**Model Creation Flow (Text to 3D or Image/Photo to 3D Screen)**:
1.  **User Action**: User is on the specific creation screen (Text to 3D or Image/Photo to 3D), provides inputs (text, image/photo, multi-view photos), selects style from available options.
2.  **Client Action**: Generates a unique `task_id` for this entire job.
3.  **Client Action (for image/photo inputs)**: Uploads the image(s) to its Supabase Storage. Gets the Supabase URL(s).
4.  **Client Action**: Creates a record in its `input_assets` Supabase table for this `task_id`, including prompt (if text), style, input type, and the Supabase URL(s) of the uploaded asset(s). Sets initial status 'pending'.
5.  **Client Action**: Shows real-time status updates on the current screen.

    **Flow A: Direct to 3D Model (Standard flow - no intermediate image generation)**:
    6.  **Client -> BFF**: Calls appropriate BFF generation endpoint (e.g., `/generate/text-to-model`, `/generate/image-to-model`) with `task_id` and necessary parameters (including Supabase URL(s) of input asset(s) from `input_assets` if applicable).
    7.  **BFF**: (Internally) Creates/updates a record in its `models` table for this `task_id` with status 'processing'. Calls Tripo AI.
    8.  **Client**: Polls `GET /tasks/{task_id}/status?service=tripo`. Workspace/current screen shows status (e.g., 'Generating Model').
    9.  **BFF (on Tripo AI completion)**: Downloads model, uploads to client's Supabase Storage, updates the record in `models` table with the final Supabase `asset_url` and status 'complete'.
    10. **Client (on seeing 'complete' status from polling or by observing `models` table)**: Retrieves the model's Supabase URL from the `models` table. Displays model in `SculptingViewerScreen`.

    **Flow B: With 2D Image Enhancement Phase (Optional workflow for enhanced results)**:
    6.  **Client -> BFF**: Calls `/generate/text-to-image` or `/generate/image-to-image` with `task_id` and parameters (including Supabase URL(s) of input asset(s) from `input_assets`).
    7.  **BFF**: (Internally) Creates record in unified `images` table for this `task_id` with `image_type: 'ai_generated'` and status 'processing'. Calls AI provider (OpenAI, Stability, or Recraft).
    8.  **Client**: Polls `GET /tasks/{task_id}/status?service={ai_provider}`. Workspace/current screen shows status (e.g., 'Generating Enhanced Image').
    9.  **BFF (on AI completion)**: Downloads image, uploads to client's Supabase Storage, updates record in `images` table with final Supabase `asset_url` and status 'complete'.
    10. **Client (on seeing 'complete' status)**: Uses the enhanced image URL from the `images` table to proceed to 3D model generation.
    11. **Client -> BFF**: Calls `/generate/image-to-model` with `task_id` and the Supabase URL of the *enhanced image* (from `images` table) as `input_image_asset_urls`.
    12. **BFF**: (Internally) Creates/updates record in `models` table for this `task_id` with status 'processing'. Calls Tripo AI.
    13. **Client**: Polls `GET /tasks/{task_id}/status?service=tripo`. Workspace/current screen shows status (e.g., 'Generating Model').
    14. **BFF (on Tripo AI completion)**: Downloads model, uploads to client's Supabase Storage, updates the record in `models` table with final Supabase `asset_url` and status 'complete'.
    15. **Client (on seeing 'complete' status)**: Retrieves model URL from `models` table. Displays model in `SculptingViewerScreen`.

    **Flow C: Multi-View Photo Capture (Image/Photo to 3D Screen with Multi-view enabled)**:
    6.  **User Action**: Selects "Multi-view" checkbox in Image/Photo to 3D screen's control panel.
    7.  **UI Response**: Interface in control panel and/or preview area updates to show 4 photo capture areas: Front, Left, Back, Right.
    8.  **User Action**: Captures or uploads photos for each view.
    9.  **Client Action**: Uploads all 4 photos to Supabase Storage, creates/updates records in `input_assets` table with all 4 asset URLs linked to the same `task_id`.
    10. **Client -> BFF**: Calls `/generate/image-to-model` with `task_id` and array of all 4 Supabase URLs in `input_image_asset_urls`.
    11. **BFF**: Processes multi-view images for enhanced 3D reconstruction via Tripo AI.
    12. **Client**: Continues with standard polling and completion flow (steps 8-10 from Flow A).

    **Flow D: 2D Image Creation and Editing (Canvas and AI Tools)**:
    6.  **User Action**: Creates or edits 2D images using canvas tools and AI enhancement features.
    7.  **Client Action**: For each creation/edit operation, creates record in unified `images` table with appropriate `image_type` ('user_sketch', 'ai_generated', 'upload').
    8.  **Client -> BFF**: Calls image enhancement endpoints like `/generate/image-to-image`, `/generate/remove-background`, `/generate/search-and-recolor` as needed.
    9.  **BFF**: Creates records in `images` table with `image_type: 'ai_generated'` and processes through appropriate AI provider.
    10. **Client**: Can use any image from the `images` table as input for 3D model generation by following Flow A step 6 onwards.

## 6. Technical Considerations

### Redux Integration Strategy

*   **Unified State Management**: Extend existing Redux store with new TypeScript slices while maintaining compatibility with existing JavaScript slices
*   **State Shape Extension**:
    ```typescript
    interface RootState {
      app: AppState;           // existing (JS) - global app state, loading messages
      editor: EditorState;     // existing (JS) - sculpting tools, brush settings
      model: ModelState;       // existing (JS) - current model, loading state
      generation: GenerationState; // NEW (TS) - API generation workflow, multi-view state
      workspace: WorkspaceState;   // NEW (TS) - user models, filtering, pagination
      auth: AuthState;         // NEW (TS) - Supabase auth state
    }

    interface GenerationState {
      currentTaskId: string | null;
      generationType: 'text' | 'image' | null;
      inputData: {
        textPrompt?: string;
        images?: PhotoAsset[];      // Single or multi-view photos
      };
      options: {
        style: StyleOption | null;
        multiView: boolean;         // Multi-view toggle state
      };
      multiViewPhotos: {
        front: PhotoAsset | null;
        left: PhotoAsset | null;
        back: PhotoAsset | null;
        right: PhotoAsset | null;
      };
      status: 'idle' | 'uploading' | 'generating' | 'polling' | 'complete' | 'error';
      error: string | null;
    }

    interface PhotoAsset {
      uri: string;
      supabaseUrl?: string;
      uploadStatus: 'pending' | 'uploading' | 'complete' | 'error';
    }
    ```
*   **Component Reusability**: Extract model picker logic from existing `ModelPickerScreen.js` into reusable TypeScript components that work with Redux
*   **Loading State Coordination**: Integrate new API polling states with existing Redux loading patterns from `appSlice.js`

### API & Data Management

*   **RTK Query Integration**: Use Redux Toolkit Query for API calls, leveraging existing Redux infrastructure
*   **Client-Side Supabase Logic**: Robust handling of Supabase operations via RTK Query with proper error handling and state management
*   **State Synchronization**: Ensure UI reflects state from client's Supabase tables and BFF polling using unified Redux store
*   **Polling Strategy**: Implement polling using RTK Query's built-in polling capabilities for real-time status updates
*   **BFF API Key Security**: Unchanged from previous considerations
*   **Asynchronous Communication**: All BFF calls and Supabase operations handled through RTK Query

## 7. Frontend-BFF API Endpoints (Summary - Client Perspective)

(Refer to `makeit3d-api.md` for full details)
*   Client sends `task_id` and Supabase URLs for inputs to endpoints like:
    *   `POST /generate/image-to-image`
    *   `POST /generate/text-to-image` (for 2D image generation)
    *   `POST /generate/text-to-model` (for 3D model generation, Tripo only)
    *   `POST /generate/image-to-model` (for 3D model from images)
    *   `POST /generate/sketch-to-image` (for 2D image from sketches)
    *   `POST /generate/remove-background` (for background removal)
    *   `POST /generate/search-and-recolor` (for object recoloring)
    *   `POST /generate/image-inpaint` (for image inpainting)
    *   `POST /generate/refine-model`
*   Client polls `GET /tasks/{task_id}/status?service={service}` for real-time AI step status.
*   Client primarily uses its own Supabase tables (`input_assets`, `images`, `models`) as the source of truth for persisted asset URLs and overall job status.
*   **Simplified Schema**: All 2D content (uploads, AI-generated, sketches) stored in unified `images` table with `image_type` field instead of separate tables.

## 8. Implementation Tasks (Structured by Priority & Dependencies)

### Phase 1: Foundation & Infrastructure Setup

#### 1.1 Critical Security & Auth Setup
*   [ ] **CRITICAL: Execute RLS fix SQL** (see SUPABASE_AUTH_SETUP.sql) - enables security for models table
*   [ ] Setup Supabase client with project config
*   [ ] Setup Supabase Auth: initialize client, configure auth providers, manage session state
*   [ ] Implement JWT token handling for BFF API calls
*   [ ] Implement deep linking for Supabase Auth

#### 1.2 TypeScript & Development Environment
*   [ ] Add TypeScript configuration (`tsconfig.json`)
*   [ ] Install TypeScript dependencies
*   [ ] Install Supabase client library (`@supabase/supabase-js`)
*   [ ] Add Redux Toolkit Query for API state management
*   [ ] Create TypeScript definitions for existing Redux slices

#### 1.3 Redux Store Extension & State Management
*   [ ] Convert existing `store.js` to `store.ts` with proper typing
*   [ ] Create `rootReducer.ts` combining existing and new slices
*   [ ] Add new TypeScript slices: `generationSlice.ts`, `workspaceSlice.ts`, `authSlice.ts`
*   [ ] Create typed Redux hooks (`useAppSelector`, `useAppDispatch`)
*   [ ] Define multi-view photo state structure in `generationSlice.ts` (ensure no sketch data)

#### 1.4 API Infrastructure (RTK Query)
*   [ ] Create `/src/api/` directory structure
*   [ ] Implement `supabaseClient.ts`
*   [ ] Implement `bffClient.ts`
*   [ ] Create RTK Query API slices (`bffApi.ts`, `supabaseApi.ts`)
*   [ ] Define TypeScript types in `/src/types/` (ensure no sketch types)

### Phase 2: Core UI Components & Reusable Elements

#### 2.1 Authentication UI Components
*   [ ] Create UI components for login, sign-up, profile management
*   [ ] Add auth state management
*   [ ] Implement protected routes
*   [ ] Add auth error handling

#### 2.2 Component Extraction & Shared Components
*   [ ] Extract `ModelPicker.tsx` from `ModelPickerScreen.js`
*   [ ] Create `ModelCard.tsx`
*   [ ] Ensure extracted components work with existing Redux model state
*   [ ] Add TypeScript interfaces for model picker props and filtering

#### 2.3 Generation Components (Base)
*   [ ] Create `/src/components/generation/StyleSelector.tsx`
*   [ ] Implement `/src/components/generation/GenerationOptions.tsx` (multiview options)
*   [ ] Add text input components for Text to 3D screen

### Phase 3: Photo Capture & Multi-View Implementation

#### 3.1 Single Photo Capture (for Image/Photo to 3D Screen)
*   [ ] Create `/src/components/generation/PhotoCapture.tsx` for single photo/image
*   [ ] Implement camera/gallery picker
*   [ ] Add photo preview and retake/replace
*   [ ] Integrate with Redux state for single photo/image storage

#### 3.2 Multi-View Photo Capture System (for Image/Photo to 3D Screen)
*   [ ] Create `/src/components/generation/MultiViewCapture.tsx`
*   [ ] Implement 4-photo capture interface (front, left, back, right)
*   [ ] Add individual photo capture states
*   [ ] Implement photo validation
*   [ ] Add photo retake/replace for each view
*   [ ] Integrate with Redux `multiViewPhotos` state

#### 3.3 Photo Upload & Storage (for Image/Photo to 3D Screen)
*   [ ] Implement Supabase Storage upload via RTK Query
*   [ ] Add upload progress indicators
*   [ ] Handle upload errors and retry
*   [ ] Create batch upload for multi-view photos
*   [ ] Update `input_assets` table with photo URL(s)

### Phase 4: Screen Implementation & User Flows

#### 4.1 Home Screen Implementation
*   [ ] Create `/src/features/model-generation/screens/HomeScreen.tsx`
*   [ ] Implement 2x2 grid: "Text to 3D", "Image/Photo to 3D", "Sculpt Model", "Color Model"
*   [ ] Ensure "Text to 3D" navigates to `TextToModelScreen.tsx`
*   [ ] Ensure "Image/Photo to 3D" navigates to `ImageToModelScreen.tsx`
*   [ ] Create `/src/components/modals/SculptModelModal.tsx` (accessed from Home)
*   [ ] Create `/src/components/modals/ColorModelModal.tsx` (accessed from Home)
*   [ ] Implement Workspace section using Redux workspace state
*   [ ] Add delete functionality for workspace items
*   [ ] Implement Public Models section

#### 4.2 Text to 3D Screen
*   [ ] Create `/src/features/model-generation/screens/TextToModelScreen.tsx`
*   [ ] Implement layout: mode-specific control panel (left) and preview area (right) for tablet/desktop
*   [ ] Implement text prompt input in control panel
*   [ ] Integrate `StyleSelector.tsx` in control panel
*   [ ] Integrate `GenerationOptions.tsx` (for concept generation) in control panel
*   [ ] Implement generation button and form validation in control panel
*   [ ] Display text prompt and concept preview (if any) in preview area

#### 4.3 Image/Photo to 3D Screen
*   [ ] Create `/src/features/model-generation/screens/ImageToModelScreen.tsx`
*   [ ] Implement layout: mode-specific control panel (left) and preview area (right) for tablet/desktop
*   [ ] Implement Image/Photo toggle in control panel
*   [ ] Integrate `PhotoCapture.tsx` (for single image/photo) and `MultiViewCapture.tsx` (if multi-view selected) in control panel / preview area as appropriate
*   [ ] Integrate `StyleSelector.tsx` in control panel
*   [ ] Integrate `GenerationOptions.tsx` (for multi-view options) in control panel
*   [ ] Implement generation button and form validation in control panel
*   [ ] Display image/photo previews in preview area

#### 4.4 Advanced Workflows & Other Screens
*   [ ] Create `/src/features/model-generation/screens/WorkspaceScreen.tsx`
*   [ ] Implement full workspace view with filtering and pagination
*   [ ] Add support for unified `images` table display alongside models

### Phase 5: Data Management & API Integration

#### 5.1 Core Data Operations
*   [ ] Implement client-side `task_id` generation
*   [ ] Implement CRUD operations for metadata tables (`input_assets`, `images`, `models`) via RTK Query
*   [ ] Integrate with Redux loading states
*   [ ] Ensure state synchronization

#### 5.2 BFF API Integration
*   [ ] Implement Text to Model API calls
*   [ ] Implement single Image/Photo to Model API calls
*   [ ] Implement multi-view Image/Photo to Model API calls
*   [ ] Add 2D image generation and enhancement API integration
*   [ ] Create polling logic using RTK Query
*   [ ] Handle API errors and retry logic

#### 5.3 Advanced State Management
*   [ ] Implement smart navigation logic using Redux state for model status (update navigation from old Creation Hub logic)
*   [ ] Add workspace filtering and search
*   [ ] Implement model status tracking
*   [ ] Add offline state handling

### Phase 6: Navigation & Integration

#### 6.1 Navigation System
*   [ ] Update `AppNavigator.js` to include `TextToModelScreen.tsx` and `ImageToModelScreen.tsx`
*   [ ] Create bridge between new screens and existing sculpting tools
*   [ ] Ensure seamless transition from generation to sculpting workflow
*   [ ] Integrate new modals with existing navigation

#### 6.2 Cross-Feature Integration
*   [ ] Integrate generation workflow with sculpting tools
*   [ ] Ensure model loading from generation to sculpting viewer
*   [ ] Add deep linking for generation workflows
*   [ ] Implement state cleanup between workflows

### Phase 7: Testing & Validation

#### 7.1 Core Functionality Testing
*   [ ] Test Text to 3D flow
*   [ ] Test Image/Photo to 3D flow (single and multi-view)
*   [ ] Test style selection and generation options (multi-view)
*   [ ] Test 2D image generation and enhancement workflows

#### 7.2 Integration Testing
*   [ ] Test with existing users
*   [ ] Verify RLS policies
*   [ ] Test auth flows
*   [ ] Test BFF integration with JWT
*   [ ] Test transition to sculpting tools

#### 7.3 UI/UX Testing
*   [ ] Test Text to 3D screen layout and controls
*   [ ] Test Image/Photo to 3D screen layout, controls, and multi-view interface
*   [ ] Validate photo capture/upload states
*   [ ] Test generation button states
*   [ ] Test modal workflows

### Phase 8: Polish & Optimization

#### 8.1 Performance Optimization
*   [ ] Optimize photo upload
*   [ ] Implement image compression
*   [ ] Add caching
*   [ ] Optimize Redux updates

#### 8.2 Error Handling & User Experience
*   [ ] Add comprehensive error messages
*   [ ] Implement retry mechanisms
*   [ ] Add loading states
*   [ ] Implement offline handling

## 9. Asset Relationships & Data Lineage

### 2D-3D Asset Linking Strategy

The simplified architecture maintains relationships between 2D images and 3D models through multiple complementary approaches:

#### Primary Linking: Task ID
- **Shared Context**: Both images and models created in the same workflow share a `task_id`
- **Query Pattern**: Find related assets by joining on `task_id`
```sql
-- Find 3D model from source image
SELECT m.* FROM models m 
WHERE m.task_id = (SELECT task_id FROM images WHERE id = 'image-id');
```

#### Direct Reference: Source Image ID
- **Models Table**: Added `source_image_id` field for direct linking
- **Use Case**: Quick lookup of the primary source image for a 3D model
```sql
-- Direct relationship lookup
SELECT i.asset_url as source_image, m.asset_url as model_url
FROM models m 
JOIN images i ON m.source_image_id = i.id
WHERE m.id = 'model-id';
```

#### Complex Workflows: Asset History
- **Multi-step Tracking**: For workflows like `upload → enhance → generate 3D`
- **Cross-table Relationships**: Links between images and models with operation context
```sql
-- Full workflow reconstruction
WITH RECURSIVE workflow AS (
  SELECT parent_asset_id, parent_asset_type, child_asset_id, child_asset_type, 
         operation_type, 1 as level
  FROM asset_history 
  WHERE child_asset_id = 'final-model-id'
  
  UNION ALL
  
  SELECT ah.parent_asset_id, ah.parent_asset_type, ah.child_asset_id, ah.child_asset_type,
         ah.operation_type, w.level + 1
  FROM asset_history ah
  JOIN workflow w ON ah.child_asset_id = w.parent_asset_id
)
SELECT * FROM workflow ORDER BY level DESC;
```

### Frontend Implementation Patterns

#### Workspace Display Strategy
- **Unified Asset View**: Display both 2D images and 3D models in a single workspace grid
- **Relationship Indicators**: Show visual connections between source images and derived models
- **Context-Aware Actions**: Present relevant actions based on asset type and relationships (e.g., "Generate 3D" for images, "View Source" for models)

#### Smart Navigation
- **Contextual Transitions**: Navigate seamlessly between related assets (model → source image, image → derived models)
- **Workflow Continuity**: Resume interrupted workflows from any point in the creation process
- **Preview Integration**: Show source images when viewing 3D models for context

#### Data Management Approach
- **Efficient Loading**: Fetch relationships on-demand using JOIN queries or separate requests based on UI needs
- **Caching Strategy**: Cache frequently accessed relationships to minimize database queries
- **Real-time Updates**: Sync asset relationships as new items are created through ongoing workflows

---
*This document reflects changes based on BFF API v1.1.0 and the agreed Supabase interaction model, with Sketch-to-3D removed and Home Screen/Creation flows updated.*