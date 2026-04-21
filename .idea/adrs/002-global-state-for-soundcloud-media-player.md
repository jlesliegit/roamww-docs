# Implementing Global State for SoundCloud Media Player
## 21/04/2026 Proposed
## Context 

Currently SoundCloud playback is tied to MixPage component. Navigating away from specific MixPage unmounts the player, 
causing the music to stop playing and in turn preventing a seamless browsing experience for the user - not allowing them 
to consume additional content across the application whilst listening to music. 

## Decision
Lift Media Player component and its associated State to a Global Context Provider Wrapped around the main App component

## Results
### Positives
Improved UX - audio persists across navigation over application

Avoid prop drilling across multiple layers by abstraction to Context 

### Negatives
Wrapping whole App component in a provider like this means that any state change can trigger a re-render 

Third-party lifecycle - abstracting to SoundCloud iFrame means that this lifecycle needs to be managed in React Context

Dealing with empty state of the player and how to manage this when no mix selected 