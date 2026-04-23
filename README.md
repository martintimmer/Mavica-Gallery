MAVICA GALLERY // Digital Archive
A retro-futuristic web gallery inspired by the Sony Mavica FD-91 floppy disk camera.

VersionLicenseStyle

🖥️ Overview
MAVICA GALLERY is a single-page web application designed to mimic the look and feel of late-90s digital camera hardware. It features a fully functional CRT monitor aesthetic, complete with scanlines, phosphor glow, and a simulated BIOS boot sequence.

This project serves as a stylish, lightweight digital archive for your photos, supporting both local storage (in-browser memory) and persistent cloud storage via Amazon S3.

Demo Screenshot(Replace the link above with an actual screenshot of your interface)

✨ Features
**沉浸式复古界面
开机启动序列: Watch the system "boot up" with a retro terminal loading screen before the interface appears.
照片显影动画: New photos "develop" on screen using a chemical-photography style animation.
FLIP 动态重组: Gallery items animate smoothly into new positions when sorting or uploading.
拖拽上传: Import photos by dragging them directly onto the floppy disk icon zone.
S3 云存储集成: Optional configuration to save images permanently to an AWS S3 bucket via presigned URLs.
响应式瀑布流: A dynamic masonry grid layout that adapts to mobile, tablet, and desktop screens.
无障碍访问: Keyboard-navigable gallery and lightbox with proper ARIA labels.
🛠️ Tech Stack
HTML5 / CSS3 (No frameworks, pure styling)
Tailwind CSS (via CDN for utility classes)
Vanilla JavaScript (ES6+)
Google Fonts: Press Start 2P & VT323
🚀 Getting Started
This is a static single-file application. No build tools or Node.js servers are required.

Download or clone the repository.
Open index.html in your web browser.
(Optional) Click the Settings (⚙️) icon to configure your S3 API endpoints.
⚙️ Configuration (S3 Storage)
By default, the gallery runs in LOCAL MODE (images are stored in browser memory and vanish on refresh). To enable persistent cloud storage:

Open the settings modal via the ⚙️ icon.
Provide your API endpoints.
Upload API Endpoint
The system expects a POST endpoint that generates a presigned S3 URL.

Sent: { filename: "image.jpg", contentType: "image/jpeg" }
Expected Response:
{  "presignedUrl": "https://s3-bucket-url...",  "fileUrl": "https://s3-bucket-url/image.jpg",  "key": "image.jpg"}
Gallery List Endpoint
A GET endpoint to populate the gallery on load.

Expected Response:
{   "images": [     { "url": "...", "filename": "photo1.jpg" },     { "url": "...", "filename": "photo2.jpg" }   ] }
(Also accepts a flat array of strings or objects with .url properties).
📂 Project Structure
/├── index.html      # The main application file (contains HTML, CSS, JS)├── README.md       # You are here└── LICENSE
🎨 Customization
You can easily tweak the color scheme by modifying the CSS variables inside the <style> tag in index.html:

:root {
    --phosphor: #00ff41; /* Primary Green Text */
    --amber: #ffb000;    /* Secondary Amber Text */
    --crt: #0a0a0a;      /* Background Black */
}

📜 License
This project is open-sourced under the MIT License.

MAVICA GALLERY v2.1 // FLOPPY DISK DIGITAL ARCHIVE
```
