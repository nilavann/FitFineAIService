# 0005. Amazon Rekognition for face matching

**Status:** Accepted

## Context
The first feature identifies gym members from a photo. This requires face
embeddings designed for identity matching and a similarity search over all
enrolled members.

## Decision
Use Amazon Rekognition face collections. Enrolment calls `IndexFaces` and
lookup calls `SearchFacesByImage`. The member identifier is stored as the
face's external image identifier, and member details are held in DynamoDB.

## Alternatives considered
- **Bedrock multimodal embeddings with a vector store:** general-purpose image
  embeddings are not trained for identity matching and would need a separate
  vector database to operate.
- **Self-hosted face model:** more control, but requires hosting, scaling, and
  index management that this service avoids.

## Consequences
- No models or vector index to operate; Rekognition holds the embeddings.
- Rekognition stores feature vectors, not images, so source photos are kept
  separately only if there is a reason to.
- Matching quality and thresholds are governed by the service; the application
  controls the similarity threshold and result count.
- Face data is biometric data and carries legal and consent obligations, covered
  in the feature document.
