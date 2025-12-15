Known Issues:
Testing:
- Even tests that don't need a structure should have an empty 1x1x1 structure, otherwise the test might randomly get stuck waiting for chunks to load.
- Teleporting a mock player in 1.21.6+ in the same game tick as they were spawned can cause the server to get stuck waiting for chunks to load.
