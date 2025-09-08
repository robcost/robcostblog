---
title: Building an Infinite Canvas with Skia in React Native
date: 2025-09-08
authors:
  - name: robcost
    link: https://github.com/robcost
tags:
  - Mobile Development
  - React Native
  - Skia
excludeSearch: false
---
Building an infinite canvas in React Native seems straightforward until you actually try to do it. You quickly run into performance issues, coordinate system headaches, and gesture conflicts that make you question your life choices. After spending way too much time figuring this out, here's everything I learned about building a proper infinite canvas that actually works.

## Why React Native Skia?

First, why not just use a ScrollView? Because ScrollView has fixed dimensions and doesn't give you the fine-grained control you need for things like:

- Custom coordinate transformations
- Efficient content culling  
- Complex gesture handling
- Drawing overlays
- Sub-pixel precision

[React Native Skia](https://shopify.github.io/react-native-skia/) gives you direct access to the Skia graphics engine with hardware acceleration. It's what Flutter uses under the hood, and it's perfect for this kind of thing.

## The Core Challenge

An infinite canvas needs to handle:

1. **Coordinate Systems** - Converting between world coordinates (your infinite space) and screen coordinates
2. **Performance** - Only rendering what's visible
3. **Gestures** - Pan, zoom, tap without conflicts
4. **Memory** - Not loading everything at once

The trick is getting all of these to work together smoothly at 60fps.

## Setting Up the Foundation

Let's start with the basic structure. You'll need these dependencies:

```bash
npm install @shopify/react-native-skia react-native-reanimated react-native-gesture-handler
```

Here's our basic canvas component:

```tsx
import React, { useCallback, useState } from 'react';
import { View, Dimensions } from 'react-native';
import {
  Canvas,
  Group,
  Rect,
  Text as SkiaText,
  matchFont
} from '@shopify/react-native-skia';
import {
  Gesture,
  GestureDetector,
} from 'react-native-gesture-handler';
import { 
  useSharedValue,
  useDerivedValue,
  withDecay
} from 'react-native-reanimated';

interface ContentItem {
  id: string;
  position: { x: number; y: number };
  size: { width: number; height: number };
  type: 'text' | 'shape';
  data: any;
}

export const InfiniteCanvas: React.FC<{
  items: ContentItem[];
  onItemSelected?: (id: string) => void;
}> = ({ items, onItemSelected }) => {
  const screenDimensions = Dimensions.get('window');
  
  // Shared values for smooth gesture handling
  const panX = useSharedValue(0);
  const panY = useSharedValue(0);
  const zoom = useSharedValue(1);
  
  // ... we'll build this out step by step
};
```

## Step 1: Transform Matrix Magic

The key insight is to use a single transform matrix instead of converting coordinates for every item. This is way more efficient:

```tsx
// Transform using a single Group transform
const transform = useDerivedValue(() => {
  return [
    { translateX: panX.value },
    { translateY: panY.value },
    { scale: zoom.value }
  ];
});

// Render content with single transform
const renderContent = () => {
  return (
    <Canvas style={{ flex: 1 }}>
      <Group transform={transform}>
        {/* Background */}
        <Rect
          x={-5000}
          y={-5000}
          width={10000}
          height={10000}
          color="#f8f9fa"
        />
        
        {/* All items render in world coordinates */}
        {items.map(item => (
          <ContentItemRenderer key={item.id} item={item} />
        ))}
      </Group>
    </Canvas>
  );
};
```

This approach means:
- Items stay in their world coordinates 
- Single transform handles all positioning
- Skia does the heavy lifting on the GPU
- 60fps performance even with hundreds of items

## Step 2: Gesture Handling Without Conflicts

Here's something I didn't think of at the start. You need different gestures for different purposes, and they can't interfere with each other.

### Multi-finger Pan (Preserves Single Finger for other actions)

```tsx
const panGesture = Gesture.Pan()
  .minPointers(2) // Only pan with 2+ fingers
  .onStart(() => {
    'worklet';
    startPanX.value = panX.value;
    startPanY.value = panY.value;
  })
  .onUpdate((event) => {
    'worklet';
    panX.value = startPanX.value + event.translationX;
    panY.value = startPanY.value + event.translationY;
  })
  .onEnd((event) => {
    'worklet';
    // Add momentum with smooth physics
    panX.value = withDecay({
      velocity: event.velocityX,
      deceleration: 0.998,
      clamp: [-10000, 10000]
    });
    
    panY.value = withDecay({
      velocity: event.velocityY,
      deceleration: 0.998,
      clamp: [-10000, 10000]
    });
  });
```

The `minPointers(2)` is crucial - it leaves single finger free for other interactions.

### Pinch-to-Zoom with Focal Point

This is the hardest part to get right. Most implementations zoom around the center, but users expect it to zoom where their fingers are:

```tsx
const focalX = useSharedValue(screenDimensions.width / 2);
const focalY = useSharedValue(screenDimensions.height / 2);
const scalePrevious = useSharedValue(1);
const scaleCurrent = useSharedValue(1);

const pinchGesture = Gesture.Pinch()
  .onStart((event) => {
    'worklet';
    // Lock focal point at gesture start
    focalX.value = event.focalX;
    focalY.value = event.focalY;
    scaleCurrent.value = 1;
  })
  .onUpdate((event) => {
    'worklet';
    const newScale = Math.max(0.5, Math.min(5.0, event.scale));
    scaleCurrent.value = newScale;
  })
  .onEnd(() => {
    'worklet';
    // Calculate pan adjustment to keep focal point fixed
    const currentScale = scalePrevious.value;
    const scaleChange = scaleCurrent.value;
    const transformedFocalX = (focalX.value - panX.value) / currentScale;
    const transformedFocalY = (focalY.value - panY.value) / currentScale;
    const panAdjustX = transformedFocalX * (1 - scaleChange) * currentScale;
    const panAdjustY = transformedFocalY * (1 - scaleChange) * currentScale;
    
    // Commit final position
    panX.value = panX.value + panAdjustX;
    panY.value = panY.value + panAdjustY;
    scalePrevious.value = scalePrevious.value * scaleCurrent.value;
    scaleCurrent.value = 1;
  });
```

The math here handles coordinate space conversion so content stays under the user's fingers during zoom. It's complex but the result feels natural.

### Tap-to-Select with Coordinate Conversion

To handle taps, you need to convert screen coordinates back to world coordinates:

```tsx
const tapGesture = Gesture.Tap()
  .onStart((event) => {
    'worklet';
    runOnJS(handleTap)(event.x, event.y, zoom.value, panX.value, panY.value);
  });

const handleTap = useCallback((
  screenX: number, 
  screenY: number, 
  currentZoom: number, 
  currentPanX: number, 
  currentPanY: number
) => {
  // Inverse transform: screen → world
  const unscaledX = screenX / currentZoom;
  const unscaledY = screenY / currentZoom;
  const worldX = unscaledX - currentPanX;
  const worldY = unscaledY - currentPanY;
  
  // Hit test against items
  const tappedItem = items.find(item => {
    return worldX >= item.position.x && 
           worldX <= item.position.x + item.size.width &&
           worldY >= item.position.y && 
           worldY <= item.position.y + item.size.height;
  });
  
  if (tappedItem && onItemSelected) {
    onItemSelected(tappedItem.id);
  }
}, [items, onItemSelected]);
```

## Step 3: Performance Optimization with Content Culling

Don't render items that aren't visible. This is essential for large datasets:

```tsx
const getVisibleItems = useCallback(() => {
  const currentZoom = scalePrevious.value * scaleCurrent.value;
  const currentPanX = panX.value;
  const currentPanY = panY.value;
  
  // Calculate viewport bounds in world coordinates
  const viewportWorldBounds = {
    left: (0 - currentPanX) / currentZoom,
    right: (screenDimensions.width - currentPanX) / currentZoom,
    top: (0 - currentPanY) / currentZoom,
    bottom: (screenDimensions.height - currentPanY) / currentZoom
  };
  
  // Add margin for smooth scrolling
  const margin = 300 / currentZoom;
  
  return items.filter(item => {
    const itemLeft = item.position.x;
    const itemRight = item.position.x + item.size.width;
    const itemTop = item.position.y;
    const itemBottom = item.position.y + item.size.height;
    
    // Check if item intersects with viewport
    return !(itemRight < viewportWorldBounds.left - margin ||
             itemLeft > viewportWorldBounds.right + margin ||
             itemBottom < viewportWorldBounds.top - margin ||
             itemTop > viewportWorldBounds.bottom + margin);
  });
}, [items, screenDimensions]);

const [visibleItems, setVisibleItems] = useState(items);

useEffect(() => {
  const interval = setInterval(() => {
    setVisibleItems(getVisibleItems());
  }, 100);
  
  return () => clearInterval(interval);
}, [getVisibleItems]);
```

## Step 4: Putting It All Together

Here's the complete gesture setup:

```tsx
const combinedGesture = Gesture.Simultaneous(
  panGesture,
  pinchGesture,
  tapGesture
);

return (
  <View style={{ flex: 1 }}>
    <GestureDetector gesture={combinedGesture}>
      <View style={{ flex: 1 }}>
        <Canvas style={{ flex: 1 }}>
          <Group transform={transform}>
            <Rect
              x={-5000}
              y={-5000}
              width={10000}
              height={10000}
              color="#f8f9fa"
            />
            
            {visibleItems.map(item => (
              <ContentItemRenderer 
                key={item.id} 
                item={item}
              />
            ))}
          </Group>
        </Canvas>
      </View>
    </GestureDetector>
  </View>
);
```

## Common Pitfalls and Solutions

### 1. Gesture Conflicts
**Problem**: Single finger gestures interfere with drawing
**Solution**: Use `minPointers(2)` for navigation gestures

### 2. Poor Zoom Experience  
**Problem**: Zoom happens at center, not at fingers
**Solution**: Implement focal point tracking with coordinate space conversion

### 3. Performance Issues
**Problem**: Rendering all items all the time
**Solution**: Implement viewport culling with margin

### 4. Coordinate Confusion
**Problem**: Getting world vs screen coordinates mixed up
**Solution**: Use single transform matrix, convert only when needed

### 5. No Momentum
**Problem**: Pan gesture stops abruptly
**Solution**: Use `withDecay()` for natural physics

## Advanced Tips

### Memory Management
For really large datasets, implement progressive loading:

```tsx
useEffect(() => {
  const viewportCenter = { 
    x: -panX.value, 
    y: -panY.value 
  };
  
  // Load items near viewport, unload distant ones
  loadItemsNearPoint(viewportCenter);
}, [/* viewport changes */]);
```

### Custom Content Rendering
Make your content renderer flexible:

```tsx
const ContentItemRenderer = ({ item }) => {
  switch (item.type) {
    case 'text':
      return (
        <SkiaText
          x={item.position.x}
          y={item.position.y}
          text={item.data.text}
          font={matchFont({ fontSize: 16 })}
          color="#000000"
        />
      );
    case 'shape':
      return (
        <Rect
          x={item.position.x}
          y={item.position.y}
          width={item.size.width}
          height={item.size.height}
          color={item.data.color}
        />
      );
    default:
      return null;
  }
};
```

### Performance Monitoring
Add debug overlay to track performance:

```tsx
const DebugOverlay = ({ visibleCount, totalCount, fps }) => (
  <Group>
    <Rect x={10} y={10} width={200} height={60} color="rgba(0,0,0,0.7)" />
    <SkiaText
      x={20}
      y={30}
      text={`Visible: ${visibleCount}/${totalCount}`}
      font={matchFont({ fontSize: 12 })}
      color="white"
    />
    <SkiaText
      x={20}
      y={50}
      text={`FPS: ${fps}`}
      font={matchFont({ fontSize: 12 })}
      color="white"
    />
  </Group>
);
```

## Key Takeaways

1. **Single Transform**: Use one transform matrix, not per-item coordinate conversion
2. **Smart Gestures**: Multi-finger for navigation, preserve single finger for other uses
3. **Focal Point Math**: The coordinate space conversion is tricky but worth getting right
4. **Cull Aggressively**: Only render what's visible (plus margin)
5. **Use Worklets**: Keep gesture handling on the UI thread for smoothness
6. **Physics Matter**: `withDecay()` makes interactions feel natural

## Performance Results

With this approach, you should get:
- 60fps smooth pan/zoom even with hundreds of items
- Natural momentum physics
- Accurate focal point zoom behavior  
- Zero gesture conflicts
- Efficient memory usage

The key insight is that React Native Skia is incredibly powerful when you work with it instead of fighting it. The single transform approach leverages GPU acceleration while the gesture handling keeps everything smooth on the UI thread.

## Useful Resources

- [React Native Skia Docs](https://shopify.github.io/react-native-skia/)
- [React Native Reanimated Docs](https://docs.swmansion.com/react-native-reanimated/)
- [React Native Gesture Handler](https://docs.swmansion.com/react-native-gesture-handler/)
- [Skia Graphics Library](https://skia.org/)

Building an infinite canvas is definitely one of those "seems simple but actually complex" problems. But once you understand the coordinate systems, gesture handling, and performance considerations, it becomes a really powerful tool for creating rich, interactive experiences in React Native.