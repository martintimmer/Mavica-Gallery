# MAVICA GALLERY // Digital Archive

> A retro-futuristic web gallery inspired by the Sony Mavica FD-91 floppy disk camera.

![Version](https://img.shields.io/badge/VERSION-2.1-green?style=flat-square&labelColor=black)
![License](https://img.shields.io/badge/LICENSE-MIT-blue?style=flat-square)
![Style](https://img.shields.io/badge/AESTHETIC-CRT%20/%20Retro-orange?style=flat-square)

---

### 🖥️ Overview

**MAVICA GALLERY** is a single-page web application designed to mimic the look and feel of late-90s digital camera hardware. It features a fully functional CRT monitor aesthetic, complete with scanlines, phosphor glow, and a simulated BIOS boot sequence.

This project serves as a stylish, lightweight digital archive for your photos, supporting both local storage (in-browser memory) and persistent cloud storage via Amazon S3.

---

### ✨ Features

- **Immersive Retro UI:** Authentic CRT scanlines, screen noise, and phosphor green glow effects.
- **Boot Sequence:** Watch the system "boot up" with a retro terminal loading screen before the interface appears.
- **Develop Animation:** New photos "develop" on screen using a chemical-photography style animation.
- **FLIP Animation:** Gallery items animate smoothly into new positions when sorting or uploading.
- **Drag & Drop:** Import photos by dragging them directly onto the floppy disk icon zone.
- **S3 Integration:** Optional configuration to save images permanently to an AWS S3 bucket via presigned URLs.
- **Responsive Masonry:** A dynamic grid layout that adapts to mobile, tablet, and desktop screens.
- **Accessibility:** Keyboard-navigable gallery and lightbox with proper ARIA labels.

---

### 🛠️ Tech Stack

- **HTML5 / CSS3** (Pure styling, no CSS files needed)
- **Tailwind CSS** (via CDN for utility classes)
- **Vanilla JavaScript** (ES6+)
- **Google Fonts:** *Press Start 2P* & *VT323*

---

### 🚀 Getting Started

This is a static single-file application. No build tools or Node.js servers are required.

1. **Download** or clone the repository.
2. Open `index.html` in your web browser.
3. (Optional) Click the **Settings** (⚙️) icon to configure your S3 API endpoints.

---

### ⚙️ Configuration (S3 Storage)

By default, the gallery runs in **LOCAL MODE** (images are stored in browser memory and vanish on refresh). To enable persistent cloud storage:

1. Open the settings modal via the ⚙️ icon.
2. Provide your API endpoints.

#### Upload API Endpoint
The system expects a `POST` endpoint that generates a presigned S3 URL.

*   **Sent:** `{ filename: "image.jpg", contentType: "image/jpeg" }`
*   **Expected Response:**
    ```json
    {
      "presignedUrl": "https://s3-bucket-url...",
      "fileUrl": "https://s3-bucket-url/image.jpg",
      "key": "image.jpg"
    }
    ```

#### Gallery List Endpoint
A `GET` endpoint to populate the gallery on load.

*   **Expected Response:**
    ```json
    { 
      "images": [ 
        { "url": "...", "filename": "photo1.jpg" }, 
        { "url": "...", "filename": "photo2.jpg" } 
      ] 
    }
    ```
    *(Also accepts a flat array of strings or objects with `.url` properties)*.

---

### 📂 Project Structure

```text
/
├── index.html      # The main application file (contains HTML, CSS, JS)
├── README.md       # You are here
└── LICENSE
```

---

### 🎨 Customization

You can easily tweak the color scheme by modifying the CSS variables inside the `<style>` tag in `index.html`:

```css
:root {
    --phosphor: #00ff41; /* Primary Green Text */
    --amber: #ffb000;    /* Secondary Amber Text */
    --crt: #0a0a0a;      /* Background Black */
}
```

---

### 📜 License

This project is open-sourced under the **MIT License**.

---

> *MAVICA GALLERY v2.1 // FLOPPY DISK DIGITAL ARCHIVE*
````
