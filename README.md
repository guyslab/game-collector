# game-collector

## Background

This is a system that collects games from various resources.

## Requirements:

1. Collect unique games from multiple sources
2. Expose API for listing collected games

## Architecture

The system is based on microservices architecture, where services communicate indirectly via message brokers for event processing, or directly by HTTP requests for queries.
Below is the model, follwoed by a list of events (messages), followed by a list of services, followed by a sequence diagram to discribe the interaction.

Collection of raw data of games is performed by scheduling web scraping jobs to scrape the sources, where each scraper is designed to scrape only one source.
A source-specific adapter is transforming each raw game to a domain model game,

Splitting the raw game processing services per source may produce duplicate source code to some extent. However, the number of source is not assumed to scale too much, and the benefit of adapting to changes in record structure of each separate source seems to worth it.

The raw games data is then processed to provide a unique set of games, to then get queried by clients.

### Model:

The following document represents the game model.

#### game

```json
{
  "start_time": <datetime>,
  "sport_type": <string>,
  "competition_name": <string>,
  "team_names": <array<string>>
}
```


### Events:

#### scrape-due

A scraping job request has been issued.

Payload:

```json
{
  "source_id": <string>,
  "created_at": <datetime>
}
```

#### raw-game

A source-specific game record has been scraped out of a source

Payload:

```json
{
  "source_id": <string>,
  <rest of structure depends on the source>
}
```

#### game

A game model record has been adapted from a source

Payload:

```json
{
  "source_id": <string>,
  "game": <game (model)>
}
```

#### unique-game

A new unique game has been collected

Payload:

```json
{
  "game": <game (model)>
}
```

### Services:

| Name                       | Functionality                                                              | Publishes   | Subscribes  | Queries/Commands                             | Database                                                               |
|----------------------------|----------------------------------------------------------------------------|-------------|-------------|----------------------------------------------|------------------------------------------------------------------------|
| scraping-scheduler         | Configures and triggers scraping jobs for a source                         | scrape-due  |             | PUT /sources/{source_id}/scraping-plan       | Document (for scraping-plans)                                          |
| scraper-<source_id>        | Performs scraping jobs at the desired source, resulting in raw game events | raw-game    | scrape-due  |                                              |                                                                        |
| source-adapter-<source_id> | Transforms source-specific game records to domain model game schema        | game        | raw-game    |                                              |                                                                        |
| game-identifier            | Ignores duplicate games by uniquely identifying them                       | unique-game | game        |                                              | Document (for unique game indexing)                                    |
| games-report               | Stores and exposes games for query                                         |             | unique-game | GET /games?page={page}&page_size={page_size} | Relational (for games dataset retrievals and perhaps further analysis) |

### Other components:

* MQTT server: raw game events message borker, to provide channel for semi-structured game data originating various external sources.
* AMQP server: message broker for domain events
* Client Gateway: Exposes the internal services to commands and queries from clients, while all other services mentioned above do not provide public access. May feature authorization, throttling, load-balancing, and other user-facing measures.

