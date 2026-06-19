# Implementing Global State for SoundCloud Media Player

## 21/04/2026
## Status: Accepted
## Context 

Currently SoundCloud playback is tied to MixPage component. Navigating away from specific MixPage unmounts the player, 
causing the music to stop playing and in turn preventing a seamless browsing experience for the user - not allowing them 
to consume additional content across the application whilst listening to music. 

## Decision
Lifted media player to a global level using React Context and SoundCloud widget API.

Global `PlayerProvider` manages state (`currentTrackId`, `isPlaying`) and holds reference to API through `useRef`

Single `<GlobalMixPlayer />` component mounts at root of app and outside of React Router `<Routes>`


## Consequences
### Positives
Improved UX - audio persists across navigation over application. Included migration of `<a>` tags to React Route `<Link>`
tags to prevent full-page browser refreshes, unmounting component

By including track_id directly in the context, I was able to avoid prop drilling track_id through multiple components 
to pass this to the iframe player

### Trade-offs
Initial concern of Provider triggering application wide re-renders mitigated by storing SoundCloud widget instance in a 
`useRef` rather than `useState`. Rely only on track_id and boolean states to trigger UI updates