# EDI 214 Integration Flow Runbook

This runbook defines a practical flow for:

1. Receive EDI 214 via SFTP
2. Generate and send 997 back to sender
3. Extract PO Number and shipment status
4. Lookup shipper UUID from internal API using PO Number
5. Branch:
   - status `X6` -> call API A
   - otherwise -> call API B

## Suggested Service Set

Create these services under:
`project.brewpack_integration.integrations`

1. `processEdi214FromSftp` (main orchestration)
2. `fetchEdi214FromSftp` (SFTP read)
3. `parseEdi214` (extract PO + status)
4. `generateAndSend997` (functional ack)
5. `lookupShipperUuidByPo` (internal API lookup)
6. `callApiA`
7. `callApiB`

## Main Orchestration Inputs/Outputs

### Input

- `sftpHost`
- `sftpPort`
- `sftpUsername`
- `sftpPassword`
- `remotePath`
- `senderId`
- `receiverId`
- `apiBaseUrl`
- `apiAuthToken`

### Output

- `success`
- `poNumber`
- `statusCode214`
- `shipperUuid`
- `apiRoute` (`A` or `B`)
- `errorMessage`

## Main Flow Logic

1. Fetch inbound EDI 214 file from SFTP.
2. Parse EDI content and extract:
   - `poNumber`
   - `statusCode214`
3. Generate and send 997 to trading partner.
4. Call internal API with `poNumber` to get `shipperUuid`.
5. Branch on `statusCode214`:
   - `X6` -> invoke `callApiA(shipperUuid, ...)`
   - `$default` -> invoke `callApiB(shipperUuid, ...)`
6. Return final status and route.

## Parsing Notes for EDI 214

Typical extraction points:

- PO Number often from `L11` segment with qualifier (implementation-specific).
- Shipment status from `AT7` segment status code.

Your partner mapping may vary. Confirm exact segment/element positions from partner spec and update parser accordingly.

## 997 Generation Notes

`generateAndSend997` should:

1. Build ACK metadata from interchange/group/transaction values in inbound 214.
2. Generate 997 payload.
3. Send back via agreed channel (SFTP/AS2/EDI transport).
4. Return `ackSent=true/false`.

## Error Handling

Recommended try/catch pattern in `processEdi214FromSftp`:

1. Catch parsing/API failures.
2. Set `success=false` and `errorMessage`.
3. Persist failure details for replay.

## Scheduler/Trigger

If no event trigger is available, schedule `processEdi214FromSftp` every 1-5 minutes.

