---
layout: post
title: "Domain-Driven Design in the Wild: Building SkySentinel's Architecture"
date: 2025-11-26
categories: [architecture, ddd, microservices, software-design]
---

# Domain-Driven Design in the Wild: Building SkySentinel's Architecture

When you're building a system that detects aircraft using acoustic analysis, correlates with flight data, and renders "storm verdicts" on position truth, traditional MVC patterns start to feel inadequate.

Enter Domain-Driven Design (DDD) - the architectural approach that transformed SkySentinel from a collection of scripts into a coherent, maintainable system.

## The Problem with Script-Driven Development

SkySentinel started simple: a Python script that measured dB levels and triggered recordings. As features grew, the codebase became unwieldy:

```python
# The monolithic approach that didn't scale
def main():
    audio_data = capture_audio()
    db_level = calculate_db(audio_data)
    if db_level > threshold:
        record_audio()
        flight_data = get_flight_data()
        correlation = correlate_audio_flight(audio_data, flight_data)
        upload_to_s3(audio_data)
        send_notification(correlation)
```

This worked for a prototype, but as the system grew more sophisticated, several problems emerged:

- **Mixed concerns**: Audio processing, flight correlation, and cloud storage all tangled together
- **Hard to test**: Business logic buried in infrastructure code
- **Difficult to extend**: Adding new features required touching multiple unrelated areas
- **No clear boundaries**: Where does audio processing end and flight correlation begin?

## Discovering the Domain

DDD starts with understanding your domain - the problem space you're solving. For SkySentinel, I identified several distinct domains:

### **Audio Domain**
- **Entities**: AudioTrigger, Recording, AcousticSignature
- **Value Objects**: DecibelLevel, Duration, Frequency
- **Services**: ThresholdCalculator, SignatureExtractor
- **Repositories**: RecordingRepository

### **Flight Domain**  
- **Entities**: Aircraft, FlightPath, Position
- **Value Objects**: Callsign, Altitude, Coordinates
- **Services**: FlightTracker, DistanceCalculator
- **Repositories**: FlightDataRepository

### **Correlation Domain**
- **Entities**: AcousticCorrelation, PositionTruth
- **Value Objects**: ConfidenceScore, GeometricValidation
- **Services**: SentinelSonarforge, CorrelationEngine
- **Repositories**: CorrelationRepository

## The Microservices Architecture

Understanding the domains led naturally to a microservices architecture:

```
SkySentinel System
├── audio-service/           # Audio Domain
│   ├── domain/             # Business logic
│   ├── infrastructure/     # External integrations
│   ├── application/        # Use cases
│   └── config/            # Service configuration
├── flight-service/         # Flight Domain  
│   ├── domain/
│   ├── infrastructure/
│   ├── application/
│   └── config/
├── sentinel-sonarforge-service/  # Correlation Domain
│   ├── domain/
│   ├── infrastructure/
│   ├── application/
│   └── config/
└── shared/                 # Common utilities
    ├── aws/
    ├── logging/
    └── config/
```

## Domain Entities in Practice

Here's how DDD principles shaped the actual code:

### **Audio Domain Entity**
```python
class AudioTrigger:
    """Core entity in the Audio domain"""
    
    def __init__(self, trigger_id: str, db_level: DecibelLevel, 
                 timestamp: datetime, location: Coordinates):
        self.trigger_id = trigger_id
        self.db_level = db_level
        self.timestamp = timestamp
        self.location = location
        self._acoustic_signature = None
    
    def extract_signature(self, extractor: SignatureExtractor) -> AcousticSignature:
        """Domain logic: how triggers get signatures"""
        if not self._acoustic_signature:
            self._acoustic_signature = extractor.extract(self)
        return self._acoustic_signature
    
    def exceeds_threshold(self, threshold: DecibelLevel) -> bool:
        """Domain logic: threshold evaluation"""
        return self.db_level.value > threshold.value
```

### **Correlation Domain Service**
```python
class SentinelSonarforge:
    """Domain service for acoustic correlation"""
    
    def hammer_position_truth(self, trigger: AudioTrigger, 
                            flight_data: List[Aircraft]) -> PositionTruth:
        """Core domain logic: forge truth from acoustic data"""
        
        # Extract acoustic fingerprint
        signature = trigger.extract_signature(self.signature_extractor)
        
        # Find correlation candidates
        candidates = self._find_correlation_candidates(signature, flight_data)
        
        # Apply geometric validation
        validated = self._apply_geometric_validation(candidates, trigger.location)
        
        # Forge the final truth
        return self._forge_position_truth(validated)
```

## The Benefits Realized

### **Clear Boundaries**
Each service has a single responsibility:
- **Audio Service**: Detect and record aircraft sounds
- **Flight Service**: Track aircraft positions and movements  
- **Sonarforge Service**: Correlate audio with flight data

### **Independent Deployment**
Services can be deployed independently:
```bash
# Deploy just the correlation improvements
cd sentinel-sonarforge-service
docker build -t sonarforge:v2.1 .
kubectl apply -f deployment.yaml
```

### **Testable Business Logic**
Domain logic is isolated from infrastructure:
```python
def test_position_truth_forging():
    # Pure domain test - no databases, no APIs
    trigger = AudioTrigger(...)
    aircraft = Aircraft(...)
    sonarforge = SentinelSonarforge()
    
    truth = sonarforge.hammer_position_truth(trigger, [aircraft])
    
    assert truth.confidence > 0.8
    assert truth.verdict == "TRUTH_FORGED"
```

### **Expressive Code**
The domain language appears directly in the code:
- `hammer_position_truth()`
- `forge_temperature`
- `storm_verdict`
- `acoustic_fingerprint`

## The Ubiquitous Language

DDD emphasizes creating a shared language between developers and domain experts. For SkySentinel, this language includes:

- **Acoustic Trigger**: A sound event that exceeds the detection threshold
- **Position Truth**: The correlated result of acoustic and flight data
- **Forge Temperature**: The confidence level of the correlation process
- **Storm Verdict**: The final assessment of correlation quality
- **Acoustic Fingerprint**: The unique signature extracted from audio

This language appears in code, documentation, and conversations, ensuring everyone speaks the same domain dialect.

## Challenges and Solutions

### **Service Communication**
**Challenge**: How do services communicate without tight coupling?
**Solution**: Event-driven architecture with domain events:

```python
class AcousticTriggerDetected(DomainEvent):
    def __init__(self, trigger: AudioTrigger):
        self.trigger = trigger
        self.timestamp = datetime.now()

# Audio service publishes events
event_bus.publish(AcousticTriggerDetected(trigger))

# Sonarforge service subscribes
@event_handler(AcousticTriggerDetected)
def handle_acoustic_trigger(event):
    # Begin correlation process
    pass
```

### **Data Consistency**
**Challenge**: Maintaining consistency across service boundaries
**Solution**: Eventual consistency with domain events and saga patterns

### **Service Discovery**
**Challenge**: Services need to find each other
**Solution**: AWS service discovery with proper health checks

## Lessons Learned

### **Start with the Domain**
Don't start with the database schema or API endpoints. Start by understanding the problem domain and modeling it accurately.

### **Embrace the Language**
If domain experts talk about "forging truth" and "storm verdicts," use that language in your code. It makes the system more understandable.

### **Boundaries Matter**
Clear service boundaries prevent the "big ball of mud" anti-pattern. Each service should have a single, well-defined responsibility.

### **Evolution Over Revolution**
SkySentinel evolved from monolith to microservices gradually. You don't need to start with perfect DDD - you can refactor toward it.

## The Result

Today, SkySentinel's architecture reflects its domain clearly:
- **Audio detection** is handled by audio experts
- **Flight tracking** is handled by aviation experts  
- **Acoustic correlation** is handled by signal processing experts

Each service can evolve independently, be tested in isolation, and be deployed without affecting others. The domain language ensures everyone understands what the system does and how it works.

DDD didn't just improve the code - it improved the entire development process.

---

*Domain-Driven Design transformed SkySentinel from a collection of scripts into a coherent system that reflects its problem domain. When your code speaks the same language as your domain experts, magic happens.*
