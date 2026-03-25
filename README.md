# Tam Quốc Match-3

🎮 **Game Match-3 Đối Kháng 1v1 Theo Chủ Đề Tam Quốc**

Một game match-3 chiến thuật với cơ chế đối kháng Player vs AI, lấy bối cảnh Tam Quốc với hệ thống tướng và tài nguyên phong phú.

## 🌟 Tính Năng Chính

### 🎯 Core Gameplay
- **Bàn chơi 8×8 chia sẻ**: Player và AI cùng chơi trên 1 board chung, có thể tranh giành tài nguyên
- **7 loại tài nguyên**: Đao Kiếm, Linh Dược, Binh Thư, Trống Trận, Quân Kỳ, Giáp Trụ, Ám Khí
- **Hệ thống tướng**: Chọn 2 tướng cùng phe (Ngụy/Thục/Ngô/Quần Hùng)
- **Combat mechanics**: Damage, HP, special skills, overdrive mode

### ⚔️ Hệ Thống Tướng
**Phe Ngụy** 🔵
- Tào Tháo - "Gian Hùng" (skill: Ma Vương Giáng Thế)
- Tư Mã Ý - "Thần Cơ" (skill: Không Thành Kế)

**Phe Thục** 🟢  
- Quan Vũ - "Vũ Thánh" (skill: Thanh Long Giảm Nguyệt)
- Trương Phi - "Hổ Tướng" (skill: Thiên Hạ Vô Song)
- Triệu Vân - "Thường Thắng" (skill: Bạch Mã Ngân Thương)
- Hoàng Trung - "Lão Tướng" (skill: Bách Bộ Xuyên Dương)

**Phe Ngô** 🔴
- Lưu Bị - "Nhân Đức" (skill: Tam Cố Thảo Lư)

**Quần Hùng** 🟣
- Mã Siêu - "Cẩm Mã Siêu" (skill: Tây Lương Thiết Kỵ)

### 🎨 UI/UX Features
- **Responsive design**: Tối ưu cho landscape mode, hiển thị overlay "Xoay ngang" khi ở portrait
- **Fixed canvas 1280×720px**: CSS scale để fit mọi màn hình
- **3-zone layout**: Left panel (Player), Center board, Right panel (Enemy)
- **Visual effects**: Smooth animations, particle effects, HP flash, combo badges
- **Chibi art style**: Phong cách Tam Quốc chibi cổ điển

### 🛠 Technical Features
- **React + TypeScript**: Modern stack
- **Vite**: Fast build tool
- **Tailwind CSS v4**: Utility-first styling
- **Web Worker AI**: Non-blocking AI computation
- **RAF Animation Pipeline**: 60fps smooth animations
- **Performance optimized**: < 10 re-renders per drag gesture

## 🚀 Quick Start

### Prerequisites
- Node.js 18+
- pnpm (recommended) or npm

### Installation

```bash
# Clone repository
git clone https://github.com/anhkhoyeu/Tam-Quoc-match-3.git
cd Tam-Quoc-match-3

# Install dependencies
pnpm install
# or
npm install

# Start development server
pnpm dev
# or
npm run dev
```

### Build for Production

```bash
pnpm build
# or
npm run build
```

## 📁 Project Structure

```
src/
├── app/
│   ├── animations/          # RAF animation pipeline
│   │   ├── AnimationPipeline.ts
│   │   └── heroFrames.ts
│   ├── components/          # React components
│   │   ├── BattleScreen.tsx
│   │   ├── HeroSelectScreen.tsx
│   │   ├── SharedBoard.tsx
│   │   └── ...
│   ├── game/               # Core game logic
│   │   ├── gameReducer.ts  # State management
│   │   ├── boardEngine.ts  # Match-3 mechanics
│   │   ├── heroData.ts     # Hero definitions
│   │   └── types.ts
│   ├── hooks/              # Custom React hooks
│   │   ├── useAiWorker.ts
│   │   ├── useAnimationPipeline.ts
│   │   └── useFrameMonitor.ts
│   └── workers/            # Web Workers
│       └── ai.worker.ts
├── imports/                # Figma assets & design specs
├── styles/                 # Global styles
│   ├── index.css
│   ├── theme.css
│   └── fonts.css
└── assets/                 # Static assets
```

## 🎮 Gameplay

### Game Flow
1. **Splash Screen** → Logo animation
2. **Main Menu** → Chọn "Bắt đầu trận chiến" hoặc "Hướng dẫn"
3. **Hero Select** → Chọn phe → Chọn 2 tướng
4. **Battle** → Đấu với AI trên board chung
5. **Game Over** → Hiển thị kết quả, "Chơi lại" hoặc "Về menu"

### Controls
- **Drag & Drop**: Kéo tile để swap
- **Tap**: Tap 2 tiles liền kề để swap
- **Túi Cẩm Nang**: Mở shop mua item boost
- **Đầu Hàng**: Xác nhận thua ngay lập tức

## 🏗 Architecture

### State Management
- **React useReducer**: Centralized state with `gameReducer.ts`
- **Synchronous reducer**: All async operations in components/hooks
- **Single source of truth**: One shared board for both players

### Performance Optimizations
- **Web Worker**: AI computation off main thread (~100-200ms)
- **RAF Pipeline**: Precompute animation steps, play smoothly, commit once at end
- **React.memo**: Memoized tile components
- **useRef for tracking**: High-frequency pointer events use refs, not state
- **startTransition**: Low-priority updates (battle logs)

### Animation System
```typescript
// Precompute all steps
const { stableBoard, steps } = buildResolvePipeline(board, from, to);

// Play pipeline (non-blocking)
pipeline.play(steps);

// Commit final state when done
pipeline.onDone(() => {
  dispatch({ type: 'COMMIT_STABLE_BOARD', board: stableBoard });
  dispatch({ type: 'END_TURN' });
});
```

## 📊 Performance Metrics

- **Target FPS**: 60fps
- **AI compute time**: 100-200ms (in Worker)
- **Swap animation**: 140ms
- **Explode animation**: 180ms  
- **Fall animation**: 220ms
- **Re-renders during drag**: < 10

## 🎨 Design System

### Colors
- **Ngụy** 🔵: `#3b82f6` (Blue)
- **Thục** 🟢: `#22c55e` (Green)
- **Ngô** 🔴: `#ef4444` (Red)
- **Quần Hùng** 🟣: `#a855f7` (Purple)
- **Gold**: `#c9a227`, `#f0c84a`
- **Parchment**: `#f5e6c4`

### Typography
- **Primary**: Be Vietnam Pro (body text)
- **Headings**: Noto Serif (titles, hero names)

## 🛠 Tech Stack

- **Framework**: React 18.3.1
- **Language**: TypeScript
- **Build Tool**: Vite 6.3.5
- **Styling**: Tailwind CSS 4.1.12
- **UI Components**: Radix UI, shadcn/ui
- **Icons**: Lucide React, MUI Icons
- **Animation**: Motion (Framer Motion)

## 📝 Recent Updates

### v1.0 (2026-03-25)
- ✅ Fixed AI "stuck thinking" bug
- ✅ Optimized RAF animation pipeline
- ✅ Implemented shared board architecture
- ✅ Integrated Figma background frames for hero cards
- ✅ Added surrender functionality with confirmation modal
- ✅ Enhanced UX with larger fonts and better spacing
- ✅ Added "Replay" button that preserves team composition

## 🐛 Known Issues

None currently! 🎉

## 🤝 Contributing

This is a personal project, but suggestions and bug reports are welcome via Issues.

## 📄 License

All rights reserved. This project uses assets designed specifically for this game.

## 🙏 Attributions

See [ATTRIBUTIONS.md](./ATTRIBUTIONS.md) for asset credits.

---

**Made with ❤️ using Figma Make**  
**© 2026 Tam Quốc Match-3**
