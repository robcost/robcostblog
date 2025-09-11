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
  withDecay,
  runOnJS
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
  const scalePrevious = useSharedValue(1);
  
  // Track previous pan position for delta calculation
  const prevPanX = useSharedValue(0);
  const prevPanY = useSharedValue(0);
  
  // Selection state
  const [selectedItemId, setSelectedItemId] = useState(null);
  
  // ... we'll build this out step by step
};
```

## Step 1: Transform Matrix Magic

Here's where things get interesting. Most people's first instinct is to convert coordinates for every single item on the canvas. You know, take each rectangle, calculate where it should be on screen, then draw it there. This works... until you have 200 items and your frame rate starts looking like a slideshow from 1995.

The breakthrough moment is realizing you can flip this around entirely. Instead of moving all your items, you move the entire coordinate system. It's like being Neo in The Matrix - don't try to bend the items, realize there is no spoon... err, individual item positioning.

Skia handles this beautifully with a single transform matrix. One transform to rule them all:

```tsx
// Transform using a single Group transform
const transform = useDerivedValue(() => {
  return [
    { translateX: panX.value },
    { translateY: panY.value },
    { scale: scalePrevious.value }
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

What makes this approach so elegant is that your items just sit there in their happy little world coordinates, completely oblivious to what's happening. A rectangle at (100, 50) is always at (100, 50) in world space. The magic happens when Skia applies that single transform matrix to everything at once on the GPU.

It's like having a really efficient stage manager - instead of telling each actor where to move individually, you just rotate the entire stage. The actors don't even know they're moving, but the audience sees everything shift perfectly. And because it's all happening in GPU-accelerated land, you get buttery smooth 60fps even with hundreds of items dancing around.

## Step 2: Gesture Handling Without Conflicts

Here's where things got... complicated. At first, I thought gesture handling would be straightforward - just detect some touches and move stuff around, right? Wrong. It turns out gestures are like roommates - they seem fine individually, but put them together and suddenly they're fighting over who gets to use the bathroom (or in this case, who gets to respond to that finger tap).

The real challenge is making sure your pan gesture doesn't hijack your zoom gesture, and your tap-to-select doesn't accidentally trigger while you're trying to pan. It's like trying to pat your head and rub your stomach while also juggling - technically possible, but you need to think it through.

### Single-finger Pan with 1:1 Tracking

The first gesture we need to nail is panning. Users expect that when they drag their finger 10 pixels, the canvas moves exactly 10 pixels. Sounds simple, but here's the gotcha: React Native Gesture Handler gives you `translationX` which is the total distance since the gesture started, not the frame-by-frame change.

If you use `translationX` directly, you get this weird acceleration effect where the faster you move your finger, the faster the canvas moves. It's like the canvas has had too much coffee. Instead, we need to track the delta between frames to get that perfect 1:1 tracking:

```tsx
// Track previous pan position for delta calculation
const prevPanX = useSharedValue(0);
const prevPanY = useSharedValue(0);

const panGesture = Gesture.Pan()
  .onStart((event) => {
    // Store the starting position
    prevPanX.value = event.translationX;
    prevPanY.value = event.translationY;
  })
  .onUpdate((event) => {
    // Calculate the change since last frame
    const deltaX = event.translationX - prevPanX.value;
    const deltaY = event.translationY - prevPanY.value;
    
    // Apply the delta to pan position
    panX.value += deltaX;
    panY.value += deltaY;
    
    // Update previous position for next frame
    prevPanX.value = event.translationX;
    prevPanY.value = event.translationY;
  })
  .onEnd((event) => {
    // Add momentum with smooth physics
    panX.value = withDecay({
      velocity: event.velocityX,
      deceleration: 0.998,
      clamp: [-5000, 5000]
    });
    
    panY.value = withDecay({
      velocity: event.velocityY,
      deceleration: 0.998,
      clamp: [-5000, 5000]
    });
  });
```

This approach gives perfect 1:1 finger tracking without acceleration or drift, plus natural momentum when you release your finger. The `withDecay` is the secret sauce that makes it feel native - when you flick the canvas, it doesn't just stop dead like hitting a brick wall. It coasts to a stop just like scrolling through your Instagram feed.

### Pinch-to-Zoom with Focal Point

Alright, buckle up. This is where we separate the professionals from the "it works on my machine" crowd. Zoom seems simple - pinch fingers together, things get smaller; spread them apart, things get bigger. But here's the kicker: users don't want things to zoom toward the center of the screen. They want to zoom toward whatever's between their fingers.

Think about it - when you're looking at a map and you pinch to zoom in on a specific street, you don't want the zoom to drift toward the center and suddenly you're looking at some random neighborhood three blocks away. That's like asking for directions to the bathroom and ending up in the kitchen - technically you moved, but it's not helpful.

The secret is calculating what point in world coordinates is currently under the user's fingers, and making sure that exact same point stays under their fingers as the zoom level changes:

```tsx
const scalePrevious = useSharedValue(1);

const pinchGesture = Gesture.Pinch()
  .onUpdate((event) => {
    // Apply zoom sensitivity for smooth control
    const zoomSensitivity = 0.05;
    const rawScale = 1 + (event.scale - 1) * zoomSensitivity;
    const newZoom = Math.max(0.1, Math.min(5.0, scalePrevious.value * rawScale));
    
    // Get current focal point (where fingers are)
    const currentFocalX = event.focalX;
    const currentFocalY = event.focalY;
    
    // Calculate what world point is under the focal point at current zoom
    const worldX = (currentFocalX - panX.value) / scalePrevious.value;
    const worldY = (currentFocalY - panY.value) / scalePrevious.value;
    
    // Update zoom and adjust pan to keep world point under focal point
    scalePrevious.value = newZoom;
    panX.value = currentFocalX - worldX * newZoom;
    panY.value = currentFocalY - worldY * newZoom;
  });
```

This approach calculates the world coordinate under your fingers and ensures it stays there as zoom changes. The result is smooth, drift-free zooming exactly where you expect. It's like having a really good camera operator who always keeps the action in frame, no matter how much you're zooming in or out.

### Tap-to-Select with Visual Feedback

Now we need to handle taps. This is where coordinate transformation gets its revenge. When someone taps the screen, the gesture gives you screen coordinates - but your items live in world coordinates. It's like someone telling you they saw a celebrity at "the coffee shop on Main Street" but not specifying which city. You need to do some translation.

The good news is that the math is just the inverse of what we're already doing. Take the screen coordinates, subtract the pan offset, divide by zoom level, and boom - you've got world coordinates. Then it's just a matter of checking which item (if any) contains that point:

```tsx
// Selection state
const [selectedItemId, setSelectedItemId] = useState(null);

const handleTap = useCallback((
  screenX: number, 
  screenY: number, 
  currentZoom: number, 
  currentPanX: number, 
  currentPanY: number
) => {
  // Convert screen coordinates to world coordinates
  const worldX = (screenX - currentPanX) / currentZoom;
  const worldY = (screenY - currentPanY) / currentZoom;
  
  // Hit test against items
  const tappedItem = items.find(item => {
    return worldX >= item.x && 
           worldX <= item.x + item.width &&
           worldY >= item.y && 
           worldY <= item.y + item.height;
  });
  
  setSelectedItemId(tappedItem ? tappedItem.id : null);
}, []);

const tapGesture = Gesture.Tap()
  .onStart((event) => {
    runOnJS(handleTap)(event.x, event.y, scalePrevious.value, panX.value, panY.value);
  });
```

Then in your render loop, apply visual feedback for selected items:

```tsx
{items.map(item => {
  const isSelected = selectedItemId === item.id;
  return (
    <Group key={item.id}>
      <Rect
        x={item.x}
        y={item.y}
        width={item.width}
        height={item.height}
        color={item.color}
        style="fill"
      />
      {/* Highlight selected items with thicker blue border */}
      <Rect
        x={item.x}
        y={item.y}
        width={item.width}
        height={item.height}
        color={isSelected ? "#007AFF" : "black"}
        style="stroke"
        strokeWidth={isSelected ? 3 : 1}
      />
    </Group>
  );
})}
```

## Step 3: Performance Optimization with Content Culling

Here's where we get to play the role of a very picky bouncer at an exclusive club. Not every item gets to be rendered - only the VIPs (Very Important Pixels) that are actually visible get past the velvet rope.

The brutal truth is that rendering 10,000 items when only 20 are visible is like cooking dinner for the entire neighborhood when only your immediate family is coming. It's wasteful, unnecessary, and your performance is going to suffer for it.

Content culling is the art of figuring out what's actually in the viewport and only rendering those items. It's like having X-ray vision that only shows you what matters:

```tsx
// Content culling using useDerivedValue for better performance
const visibleItems = useDerivedValue(() => {
  const currentZoom = scalePrevious.value;
  const currentPanX = panX.value;
  const currentPanY = panY.value;
  
  // Calculate viewport bounds in world coordinates
  const viewportLeft = (0 - currentPanX) / currentZoom;
  const viewportRight = (screenWidth - currentPanX) / currentZoom;
  const viewportTop = (0 - currentPanY) / currentZoom;
  const viewportBottom = (screenHeight - currentPanY) / currentZoom;
  
  // Add margin for smooth scrolling
  const margin = 300 / currentZoom;
  
  return items.filter(item => {
    const itemLeft = item.x;
    const itemRight = item.x + item.width;
    const itemTop = item.y;
    const itemBottom = item.y + item.height;
    
    // Check if item intersects with viewport
    return !(itemRight < viewportLeft - margin ||
             itemLeft > viewportRight + margin ||
             itemBottom < viewportTop - margin ||
             itemTop > viewportBottom + margin);
  });
});
```

## Step 4: Putting It All Together

Alright, it's showtime! We've got our individual players - the transform matrix is our stage, the gestures are our choreography, and the culling is our lighting director. Now we need to get them all working together like a well-oiled machine.

The key is using `Gesture.Simultaneous()` to tell React Native Gesture Handler that these gestures should play nice together. It's like being the diplomatic parent who tells the kids they all have to share the playground:

```tsx
const combinedGesture = Gesture.Simultaneous(panGesture, pinchGesture, tapGesture);

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
            
            {visibleItems.value.map(item => (
              <Group key={item.id}>
                <Rect
                  x={item.x}
                  y={item.y}
                  width={item.width}
                  height={item.height}
                  color={item.color}
                />
              </Group>
            ))}
          </Group>
        </Canvas>
      </View>
    </GestureDetector>
  </View>
);
```

## Common Pitfalls and Solutions

### 1. Pan Acceleration and Drift
**Problem**: Using `event.translationX` directly causes acceleration
**Solution**: Track delta between frames using previous position

### 2. Poor Zoom Experience  
**Problem**: Zoom happens at center, not at fingers, or causes drift
**Solution**: Apply zoom transformation continuously during gesture, not just at end

### 3. Performance Issues
**Problem**: Rendering all items all the time
**Solution**: Use `useDerivedValue` for efficient viewport culling

### 4. Coordinate Confusion
**Problem**: Getting world vs screen coordinates mixed up
**Solution**: Use single transform matrix, convert only when needed

### 5. Zoom Sensitivity
**Problem**: Pinch zoom is too fast or sensitive
**Solution**: Apply sensitivity multiplier to scale changes (e.g., 0.05 for slow zoom)

### 6. No Visual Feedback
**Problem**: Users can't tell what they've selected
**Solution**: Apply visual changes (border color/thickness) to selected items

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

To see a working example checkout my demo on Github here: [react-native-skia-infinite-canvas](https://github.com/robcost/react-native-skia-infinite-canvas)