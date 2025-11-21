# Component: Database

The Database component is responsible for all data persistence. It will be implemented as a service that other components can interact with via a well-defined API. This component will use a local SQLite database for simplicity and ease of deployment.

## Responsibilities

-   Store and retrieve chunk metadata.
-   Store and retrieve evidence data.
-   Store and retrieve identity information.
-   Store and retrieve system health metrics.
-   Store and retrieve application configuration.
-   Provide methods for querying data based on various criteria.

## Function Contracts

| Function Signature                      | Description                                                                                             | Estimated LOC |
| --------------------------------------- | ------------------------------------------------------------------------------------------------------- | ------------- |
| `__init__(db_path)`                     | Initializes the database connection. `db_path` is the path to the SQLite database file.                   | 10            |
| `create_tables()`                       | Creates the necessary tables in the database if they don't already exist.                               | 50            |
| `get_chunk(chunk_id)`                   | Retrieves a chunk by its ID. Returns a dictionary representing the chunk, or `None` if not found.         | 15            |
| `get_chunks_by_timestamp(start, end)`   | Retrieves all chunks within a given time range. Returns a list of chunk dictionaries.                   | 20            |
| `save_chunk(chunk_data)`                | Saves a new chunk to the database. Returns the ID of the newly created chunk.                           | 20            |
| `update_chunk(chunk_id, chunk_data)`    | Updates an existing chunk with new data.                                                                | 20            |
| `get_evidence(evidence_id)`             | Retrieves evidence by its ID. Returns a dictionary representing the evidence, or `None` if not found.     | 15            |
| `get_evidence_for_chunk(chunk_id)`      | Retrieves all evidence associated with a specific chunk. Returns a list of evidence dictionaries.       | 20            |
| `save_evidence(evidence_data)`          | Saves new evidence to the database. Returns the ID of the newly created evidence.                       | 20            |
| `update_evidence(evidence_id, data)`    | Updates existing evidence with new data.                                                                | 20            |
| `get_identity(identity_id)`             | Retrieves an identity by its ID. Returns a dictionary representing the identity, or `None` if not found.  | 15            |
| `get_identities(identity_ids)`          | Retrieves multiple identities by their IDs. Returns a list of identity dictionaries.                    | 20            |
| `save_identity(identity_data)`          | Saves a new identity to the database. Returns the ID of the newly created identity.                     | 20            |
| `update_identity(identity_id, data)`    | Updates an existing identity with new data.                                                             | 20            |
| `get_system_health()`                   | Retrieves the latest system health status. Returns a dictionary of health metrics.                      | 15            |
| `save_system_health(health_data)`       | Saves a new system health status snapshot.                                                              | 15            |
| `get_config()`                          | Retrieves the application configuration. Returns a dictionary of configuration settings.                | 15            |
| `save_config(config_data)`              | Saves the application configuration.                                                                    | 15            |
| `get_unprocessed_chunks()`              | Retrieves all chunks that have not yet been processed by the `ChunkProcessor`.                          | 20            |
| `get_unlinked_chunks()`                 | Retrieves all chunks that have not yet been temporally linked.                                          | 20            |

**Total Estimated Lines of Code:** 385
