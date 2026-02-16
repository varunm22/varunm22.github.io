---
title: "Virtual Aquarium: Next Steps"
date: 2026-02-15
cover:
  image: /projects/betta_pixel.png
  hidden: true
---

Development is currently paused, but here's a list of future features I'd like to add next.

## Water Chemistry

One of the important lessons for aquarists is to ensure they don't overstock or overfeed, especially early as the nitrogen cycle is being established. If we encoded the nitrogen cycle here, we could track rough concentrations of ammonia, nitrite, and nitrate. For nitrifying bacteria, we'd just track a general level that increases over time to indicate tank maturity. The level of these would control the conversion from ammonia to nitrite to nitrate.

For creation of ammonia, we could of course simply use the number/size of fish and snails. More interesting would be the consumption of nitrates by algal growth. If we scale algal growth to nitrate level, this would also reflect true aquariums. One tricky point could be balancing the consumption of nitrates by algal growth with the increased production of ammonia by the additional snails born due to that same algal consumption. It's worth thinking more about what the steady state behavior here should be, given this all operates on a much faster time scale than reality.

The affects of high nitrates (or especially nitrite and ammonia) could also be used to affect fish and snail health, making it possible for fish to become sluggish and unhealthy or even die like snails can. This would also have to be tuned in a way where it would add a fun layer of complexity without making the project too unstable to be entertaining.

## Plants

If water chemistry is implemented, another step could be to add larger plants besides just algae. The growth and trimming of plants could provide a means of further establishing a balanced clean aquarium, reflecting reality. There are some tricky portions to implementing these well.

### Graphics/growth

How should plant sprites/graphics represent growth? If we were to add floating plants like *salvinia minima*, we'd potentially want to have modular sprites per leaf and for the central stem such that plants could add leaves until they get big enough to naturally split. Then we'd also want to make sure they naturally distribute across the water surface to not overlap until they become overcrowded, at which point they could vertically stack.

Another plant we could add would be guppy grass. For this, we could have stems either planted or floating in the water column. The sprite would need to represent a single segment of stem and leaves so we could add on additional ones to represent growth, making sure to curve away from the edges of the tank.

### Collision Avoidance

Should plants represent a surface to fish and snails? For fish, this would entail having some sort of detection and avoidance of plants. They already have some form of this for walls, but it isn't necessarily super strong. For plants, this feels relevant because they would be static components in the tank, so it'd be more obvious when fish and snails pass through them. At the moment, they only pass through each other, which isn't too bad looking.

For snails, it'd be very cool to make the stem plants themselves into surfaces that the snails could crawl on. This would make the existing path finding much harder, and I'd have to figure out if algae should grow on them to motivate crawling on them in the first place.

## Betta Fish

The largest share of both logic complexity and time spent on this project was on developing the behavior of the ember tetras. A natural next step could be to add another species of fish! To have maximally different behavior patterns, I'd either go for something with more hunting instincts like a betta or a bottom dweller like a corydora. Given I already have snails, I'd lean betta, especially given the potential for fun new emergent behavior!

Some fun ideas could include:
- the betta could hunt snails and other fish: for smaller snails, it could actually successfully eat them, causing them to fall to the bottom as empty shells. For larger ones, it would just cause them to retreat into the shell, then come out again later. For ember tetras, they would hopefully just increase their fear level and flee.
- the betta could have a more complex movement system which allows it to hover/turn in place and move both forward and backward instead of having movement tied to the direction it's facing. This would give much more dynamic behavior around investigating and pursuing other creatures
- the betta could show "excitement" around feeding, following the mouse when hungry and even glass surfing when the user clicks into the "actions" tab where the feeding button is.
