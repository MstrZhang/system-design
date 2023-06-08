# uber

design a ride-sharing (logistics) service similar to uber / lyft

- user facing app that requests rides
- driver facing app that assigns riders to drivers

## estimations

use cases:

- rider can request a ride to a location
- driver can poll for rides around their area
- app provides some sort of map to display the ride
- app provides payment service:
  - rider pays for ride
  - driver receives payment for ride
- app provides some sort of routing service to show fastest route to destination
- app prices rides depending on some sort of criteria
- app provides real-time information about the status of the ride

out of scope:

- algorithms for ETA service
  - services out there like mapbox and google maps provide this
- mapping service
  - plenty of services out there like mapbox and google maps
- payments service
  - can use something like stripe to handle payments
- data analytics

## constraints and assumptions

general:

- traffic is not evenly distributed
  - some areas will have no traffic at all and some areas will have tons of traffic
    - e.g. metropolitan area vs rural area
  - traffic depends on time
    - e.g. late night on a weekend or after a concert
- status of the ride is provided in realtime
  - i.e. routing, tracking location
- service needs to handle a large number of users reading and writing at the same time

map:

- map information needs to be realtime
- map information needs to be available but can tolerate some inconsistency
  - if the user loses connection, we can catch them up when they regain connection

payments:

- payment system needs to be consistent
  - it is important that the rider's money goes to the driver properly
  - if the operation doesn't work it should be voided

ride matching service:

- ride matching should be very fast
- need a quick and easy way to match riders and drivers together
  - need location information and nearness information

rider / driver API:

- we need a realtime stream of information
- using HTTP queries is probably insufficient
  - latency issue for polling
  - polling is expensive
  - concurrency issues
  - lots of load on the rider APIs to have to poll for driver information constantly
- looks to be an obvious case for web sockets
  - rider and driver API will be in web sockets

## algorithm details

uber h3:

- idea:
  - given the location of users and location of drivers, we need a way to find drivers who are close to our rider
  - there should be some way of a subset of the globe so that we can find drivers in the same chunk as the given rider
- how h3 works:
  - splits the world into hexagons
  - given the `lng-lat` of a rider or driver, returns the `hexagon` that the `lng-lat` belongs to
  - given a `hexagon`, returns nearby hexagons
- how matching will work:
  - rider requests a ride from their given location
  - h3 calculates the `hexagon` that the rider belongs in
  - h3 finds the drivers that belong to the same `hexagon` as the user
    - use some sort of ETA service (i.e. some sort of path-finding service) to determine the closest drivers to the user
    - most mapping services will provide this
  - if there are no drivers, recurse on neighboring `hexagon`s
- uber h3 is a bit of an implementation detail but you should be able to intuitively ponder on an API
  - there needs to be some way of structuring driver data so that given a new `lng-lat`, we can find nearby drivers
  - naive idea:
    - split the map into a grid
    - perform intersection calculations when we get a driver location to determine what "square" the driver is in
      - squares don't work really well because the world is a globe (not flat)
      - hexagons also have some advantages in nearest-neighbor calculations (i.e. matching for the ETA service for things like surge; uber has a good talk on h3 and the motivations for making it)
