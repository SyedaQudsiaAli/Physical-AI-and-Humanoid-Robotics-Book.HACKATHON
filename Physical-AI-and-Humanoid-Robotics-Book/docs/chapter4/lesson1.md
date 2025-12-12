---
sidebar_position: 1
---

# Lesson 4.1: Voice-to-Text Intake using Whisper

This lesson covers the integration of Whisper for voice command transcription in humanoid robot systems, providing a comprehensive understanding of voice processing, audio preprocessing, and natural language command interpretation for effective human-robot interaction.

## Introduction

Voice-to-text transcription is a critical component of human-robot interaction that enables natural communication between humans and humanoid robots. OpenAI's Whisper model provides state-of-the-art speech recognition capabilities that can be integrated into robotic systems for robust voice command processing. This lesson explores the implementation of Whisper for voice command transcription, including audio preprocessing, noise reduction, and integration with ROS 2 for command routing and execution in humanoid robot systems.

## Key Concepts

- Whisper for voice command transcription with multiple model sizes and configurations
- Advanced audio preprocessing including noise suppression and voice activity detection
- Integration with ROS 2 for real-time command routing and processing
- Privacy and security considerations for voice data processing
- Speaker identification and voice activity detection in multi-user environments
- Real-time audio streaming and buffer management for robotic applications

## Advanced Whisper Integration Architecture

Whisper can be integrated into robotic systems with advanced preprocessing and post-processing capabilities for robust voice command transcription in challenging environments.

```python
import whisper
import rospy
import numpy as np
import pyaudio
import webrtcvad
from std_msgs.msg import String
from sensor_msgs.msg import AudioData
import threading
import queue
import time
from collections import deque
import torch
import torchaudio
from typing import Optional, Dict, Any

class AdvancedVoiceCommandNode:
    def __init__(self):
        rospy.init_node('advanced_voice_command_node')

        # Publishers for different command types
        self.command_pub = rospy.Publisher('transcribed_command', String, queue_size=10)
        self.intent_pub = rospy.Publisher('command_intent', String, queue_size=10)
        self.status_pub = rospy.Publisher('voice_status', String, queue_size=10)

        # Whisper model configuration
        self.model_size = rospy.get_param('~model_size', 'base')
        self.model = whisper.load_model(self.model_size)

        # Audio processing parameters
        self.sample_rate = 16000
        self.chunk_size = 1024
        self.audio_buffer = deque(maxlen=48000)  # 3 seconds buffer

        # Voice activity detection
        self.vad = webrtcvad.Vad(2)  # Aggressive VAD level

        # Audio stream setup
        self.audio = pyaudio.PyAudio()
        self.stream = self.audio.open(
            format=pyaudio.paInt16,
            channels=1,
            rate=self.sample_rate,
            input=True,
            frames_per_buffer=self.chunk_size
        )

        # Threading for real-time processing
        self.audio_queue = queue.Queue()
        self.processing_thread = threading.Thread(target=self.process_audio_stream)
        self.transcription_thread = threading.Thread(target=self.transcribe_audio_stream)

        # Processing flags
        self.is_listening = False
        self.is_processing = False

        # Voice activity detection parameters
        self.voice_activity_threshold = 0.5
        self.silence_duration_threshold = 1.0  # seconds
        self.min_voice_duration = 0.5  # minimum voice activity to process

        # Start processing threads
        self.processing_thread.start()
        self.transcription_thread.start()

        rospy.loginfo("Advanced Voice Command Node initialized")

    def start_listening(self):
        """Start the voice command listening process"""
        self.is_listening = True
        self.status_pub.publish(String(data="listening"))

    def stop_listening(self):
        """Stop the voice command listening process"""
        self.is_listening = False
        self.status_pub.publish(String(data="stopped"))

    def detect_voice_activity(self, audio_chunk: bytes) -> bool:
        """Detect voice activity in the audio chunk using WebRTC VAD"""
        try:
            # Convert bytes to numpy array
            audio_array = np.frombuffer(audio_chunk, dtype=np.int16)

            # Normalize audio to float32
            audio_float = audio_array.astype(np.float32) / 32768.0

            # Apply VAD - WebRTC expects 16kHz, 8kHz, or 32kHz
            vad_result = self.vad.is_speech(
                audio_chunk,
                sample_rate=self.sample_rate
            )

            return vad_result
        except Exception as e:
            rospy.logerr(f"Error in voice activity detection: {e}")
            return False

    def preprocess_audio(self, audio_data: np.ndarray) -> np.ndarray:
        """Apply noise reduction and audio preprocessing"""
        try:
            # Apply noise reduction using torchaudio
            if torch.cuda.is_available():
                audio_tensor = torch.from_numpy(audio_data).cuda()
            else:
                audio_tensor = torch.from_numpy(audio_data)

            # Apply noise reduction (simple spectral gating approach)
            # In practice, you might use more sophisticated noise reduction
            audio_tensor = torchaudio.functional.vad(audio_tensor, self.sample_rate)

            # Convert back to numpy if needed
            if audio_tensor.is_cuda:
                processed_audio = audio_tensor.cpu().numpy()
            else:
                processed_audio = audio_tensor.numpy()

            return processed_audio
        except Exception as e:
            rospy.logerr(f"Error in audio preprocessing: {e}")
            return audio_data  # Return original if preprocessing fails

    def process_audio_stream(self):
        """Continuously capture and process audio stream"""
        while not rospy.is_shutdown():
            if self.is_listening:
                try:
                    # Read audio chunk
                    audio_chunk = self.stream.read(self.chunk_size, exception_on_overflow=False)

                    # Detect voice activity
                    has_voice = self.detect_voice_activity(audio_chunk)

                    if has_voice:
                        # Add to audio buffer for processing
                        audio_array = np.frombuffer(audio_chunk, dtype=np.int16).astype(np.float32) / 32768.0
                        self.audio_buffer.extend(audio_array)

                        # Publish voice activity status
                        self.status_pub.publish(String(data="voice_detected"))
                    else:
                        # Check if we have accumulated enough voice data to process
                        if len(self.audio_buffer) > self.sample_rate * 0.5:  # At least 0.5 seconds
                            # Add to processing queue
                            audio_data = np.array(self.audio_buffer)
                            self.audio_queue.put(audio_data.copy())

                            # Clear buffer
                            self.audio_buffer.clear()

                            # Publish silence status
                            self.status_pub.publish(String(data="processing"))
                        else:
                            # Clear buffer if it's been too long without voice
                            if len(self.audio_buffer) > self.sample_rate * 3:  # 3 seconds
                                self.audio_buffer.clear()
                except Exception as e:
                    rospy.logerr(f"Error in audio stream processing: {e}")

            time.sleep(0.01)  # 10ms sleep

    def transcribe_audio_stream(self):
        """Continuously transcribe audio from the queue"""
        while not rospy.is_shutdown():
            try:
                # Get audio data from queue
                audio_data = self.audio_queue.get(timeout=1.0)

                if len(audio_data) > 0:
                    # Preprocess audio
                    processed_audio = self.preprocess_audio(audio_data)

                    # Transcribe using Whisper
                    result = self.model.transcribe(processed_audio, fp16=torch.cuda.is_available())
                    transcription = result["text"].strip()

                    if transcription:
                        # Publish transcribed command
                        self.command_pub.publish(String(data=transcription))

                        # Extract intent and publish
                        intent = self.extract_intent(transcription)
                        self.intent_pub.publish(String(data=intent))

                        rospy.loginfo(f"Transcribed: {transcription}")
                        rospy.loginfo(f"Intent: {intent}")

                        # Reset status
                        self.status_pub.publish(String(data="command_received"))
            except queue.Empty:
                continue
            except Exception as e:
                rospy.logerr(f"Error in transcription: {e}")

    def extract_intent(self, text: str) -> str:
        """Extract intent from transcribed text"""
        text_lower = text.lower()

        # Simple intent classification based on keywords
        if any(keyword in text_lower for keyword in ["move", "walk", "go", "forward", "backward", "left", "right"]):
            return "movement_command"
        elif any(keyword in text_lower for keyword in ["stop", "halt", "pause"]):
            return "stop_command"
        elif any(keyword in text_lower for keyword in ["pick", "grab", "take", "lift"]):
            return "manipulation_command"
        elif any(keyword in text_lower for keyword in ["dance", "jump", "kick"]):
            return "action_command"
        elif any(keyword in text_lower for keyword in ["follow", "come", "here"]):
            return "navigation_command"
        else:
            return "unknown_command"

    def shutdown(self):
        """Clean shutdown of audio resources"""
        self.is_listening = False
        self.stream.stop_stream()
        self.stream.close()
        self.audio.terminate()
        rospy.loginfo("Advanced Voice Command Node shutdown complete")

def main():
    voice_node = AdvancedVoiceCommandNode()

    try:
        voice_node.start_listening()
        rospy.spin()
    except KeyboardInterrupt:
        rospy.loginfo("Shutdown requested by user")
    finally:
        voice_node.shutdown()

if __name__ == '__main__':
    main()
```

## Audio Preprocessing and Noise Reduction

Effective voice command processing in robotic environments requires sophisticated audio preprocessing to handle background noise, mechanical sounds, and acoustic challenges.

```python
import numpy as np
from scipy import signal
import librosa
import webrtcvad
from typing import Tuple

class AudioPreprocessor:
    def __init__(self, sample_rate: int = 16000):
        self.sample_rate = sample_rate
        self.vad = webrtcvad.Vad(2)

        # Noise reduction parameters
        self.noise_floor_db = -40
        self.speech_threshold_db = -20

    def apply_noise_reduction(self, audio_data: np.ndarray) -> np.ndarray:
        """Apply noise reduction using spectral subtraction"""
        # Convert to frequency domain
        stft = librosa.stft(audio_data)
        magnitude = np.abs(stft)
        phase = np.angle(stft)

        # Estimate noise floor
        noise_floor = np.mean(magnitude, axis=1, keepdims=True) * 0.1

        # Apply spectral subtraction
        enhanced_magnitude = np.maximum(magnitude - noise_floor, 0)

        # Convert back to time domain
        enhanced_stft = enhanced_magnitude * np.exp(1j * phase)
        enhanced_audio = librosa.istft(enhanced_stft)

        return enhanced_audio.astype(np.float32)

    def apply_bandpass_filter(self, audio_data: np.ndarray) -> np.ndarray:
        """Apply bandpass filter to focus on human speech frequencies"""
        # Design bandpass filter for human speech (300Hz - 3400Hz)
        low_freq = 300
        high_freq = 3400

        nyquist = self.sample_rate / 2
        low = low_freq / nyquist
        high = high_freq / nyquist

        b, a = signal.butter(4, [low, high], btype='band')
        filtered_audio = signal.filtfilt(b, a, audio_data)

        return filtered_audio.astype(np.float32)

    def detect_silence(self, audio_data: np.ndarray, threshold_db: float = -30) -> bool:
        """Detect silence based on audio power"""
        power_db = 20 * np.log10(np.sqrt(np.mean(audio_data ** 2)) + 1e-10)
        return power_db < threshold_db

    def normalize_audio(self, audio_data: np.ndarray) -> np.ndarray:
        """Normalize audio to optimal range for Whisper"""
        # Normalize to -6dB RMS
        rms = np.sqrt(np.mean(audio_data ** 2))
        target_rms = 10 ** (-6 / 20)  # -6 dB in linear scale

        if rms > 0:
            gain = target_rms / rms
            normalized_audio = audio_data * gain
            # Ensure we don't exceed [-1, 1] range
            normalized_audio = np.clip(normalized_audio, -1.0, 1.0)
        else:
            normalized_audio = audio_data

        return normalized_audio

    def preprocess_for_whisper(self, audio_data: np.ndarray) -> np.ndarray:
        """Complete preprocessing pipeline for Whisper input"""
        # Apply bandpass filter
        filtered_audio = self.apply_bandpass_filter(audio_data)

        # Apply noise reduction
        denoised_audio = self.apply_noise_reduction(filtered_audio)

        # Normalize
        normalized_audio = self.normalize_audio(denoised_audio)

        # Resample if needed (Whisper expects 16kHz)
        if self.sample_rate != 16000:
            normalized_audio = librosa.resample(
                normalized_audio,
                orig_sr=self.sample_rate,
                target_sr=16000
            )

        return normalized_audio
```

## Voice Activity Detection and Speaker Identification

Advanced voice processing systems require voice activity detection and speaker identification to handle multi-user environments effectively.

```python
import numpy as np
from sklearn.mixture import GaussianMixture
from sklearn.preprocessing import StandardScaler
import librosa
from typing import List, Optional

class VoiceActivityDetector:
    def __init__(self, sample_rate: int = 16000):
        self.sample_rate = sample_rate
        self.frame_length = 25  # ms
        self.frame_step = 10    # ms
        self.vad_threshold = 0.5

    def extract_features(self, audio_data: np.ndarray) -> np.ndarray:
        """Extract features for voice activity detection"""
        # Calculate frame length and step in samples
        frame_length_samples = int(self.frame_length * self.sample_rate / 1000)
        frame_step_samples = int(self.frame_step * self.sample_rate / 1000)

        # Extract MFCC features
        mfccs = librosa.feature.mfcc(
            y=audio_data,
            sr=self.sample_rate,
            n_mfcc=13,
            n_fft=frame_length_samples,
            hop_length=frame_step_samples
        )

        # Extract spectral features
        spectral_centroids = librosa.feature.spectral_centroid(
            y=audio_data,
            sr=self.sample_rate,
            n_fft=frame_length_samples,
            hop_length=frame_step_samples
        )

        # Combine features
        features = np.vstack([mfccs, spectral_centroids])
        return features.T

    def detect_voice_activity(self, audio_data: np.ndarray) -> np.ndarray:
        """Detect voice activity in audio data"""
        features = self.extract_features(audio_data)

        # Calculate energy-based voice activity
        frame_length_samples = int(self.frame_length * self.sample_rate / 1000)
        frames = librosa.util.frame(audio_data, frame_length=frame_length_samples, hop_length=frame_length_samples//2)
        energy = np.array([np.sum(np.abs(frame)**2) for frame in frames.T])

        # Normalize energy
        energy = (energy - np.min(energy)) / (np.max(energy) - np.min(energy) + 1e-8)

        # Apply threshold
        vad = energy > self.vad_threshold

        return vad

class SpeakerIdentifier:
    def __init__(self, sample_rate: int = 16000):
        self.sample_rate = sample_rate
        self.speakers = {}  # Dictionary to store speaker models
        self.scaler = StandardScaler()

    def extract_speaker_features(self, audio_data: np.ndarray) -> np.ndarray:
        """Extract speaker identification features"""
        # Extract MFCC features for speaker identification
        mfccs = librosa.feature.mfcc(
            y=audio_data,
            sr=self.sample_rate,
            n_mfcc=20,
            n_fft=2048,
            hop_length=512
        )

        # Extract spectral rolloff, zero crossing rate, and chroma features
        spectral_rolloff = librosa.feature.spectral_rolloff(y=audio_data, sr=self.sample_rate)
        zcr = librosa.feature.zero_crossing_rate(audio_data)
        chroma = librosa.feature.chroma_stft(y=audio_data, sr=self.sample_rate)

        # Combine features
        features = np.vstack([
            np.mean(mfccs, axis=1),
            np.mean(spectral_rolloff, axis=1),
            np.mean(zcr, axis=1),
            np.mean(chroma, axis=1)
        ])

        return features.flatten()

    def enroll_speaker(self, speaker_id: str, audio_data: np.ndarray):
        """Enroll a new speaker"""
        features = self.extract_speaker_features(audio_data)

        # Train GMM model for this speaker
        gmm = GaussianMixture(n_components=2, random_state=42)
        features_reshaped = features.reshape(1, -1)

        # For real implementation, you'd need multiple samples per speaker
        # This is a simplified version
        gmm.fit(features_reshaped)
        self.speakers[speaker_id] = gmm

    def identify_speaker(self, audio_data: np.ndarray) -> Optional[str]:
        """Identify the speaker in the audio data"""
        if not self.speakers:
            return None

        features = self.extract_speaker_features(audio_data)
        features_reshaped = features.reshape(1, -1)

        best_speaker = None
        best_score = float('-inf')

        for speaker_id, model in self.speakers.items():
            score = model.score(features_reshaped)
            if score > best_score:
                best_score = score
                best_speaker = speaker_id

        return best_speaker
```

## Privacy and Security Considerations

Voice processing in robotic systems must address privacy and security concerns, especially when handling sensitive voice data.

```python
import hashlib
import hmac
import base64
from cryptography.fernet import Fernet
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.kdf.pbkdf2 import PBKDF2HMAC
import os
from typing import Optional

class VoicePrivacyManager:
    def __init__(self):
        self.encryption_key = self._generate_key()
        self.cipher = Fernet(self.encryption_key)

    def _generate_key(self) -> bytes:
        """Generate encryption key for voice data"""
        password = os.urandom(32)  # Random password
        salt = os.urandom(16)

        kdf = PBKDF2HMAC(
            algorithm=hashes.SHA256(),
            length=32,
            salt=salt,
            iterations=100000,
        )
        key = base64.urlsafe_b64encode(kdf.derive(password))
        return key

    def encrypt_audio_data(self, audio_data: bytes) -> bytes:
        """Encrypt audio data before processing"""
        return self.cipher.encrypt(audio_data)

    def decrypt_audio_data(self, encrypted_data: bytes) -> bytes:
        """Decrypt audio data for processing"""
        return self.cipher.decrypt(encrypted_data)

    def anonymize_voice_data(self, audio_data: np.ndarray) -> np.ndarray:
        """Apply voice anonymization techniques"""
        # Simple pitch shifting to anonymize voice
        # In practice, more sophisticated techniques would be used
        pitch_shift = np.random.uniform(-2, 2)  # Random pitch shift
        sr = 16000  # Sample rate

        # Apply pitch shifting using librosa
        try:
            import librosa
            anonymized_audio = librosa.effects.pitch_shift(
                audio_data,
                sr=sr,
                n_steps=pitch_shift
            )
            return anonymized_audio
        except ImportError:
            # If librosa is not available, return original
            return audio_data

    def hash_voice_features(self, features: np.ndarray) -> str:
        """Hash voice features to prevent identification"""
        feature_bytes = features.tobytes()
        feature_hash = hashlib.sha256(feature_bytes).hexdigest()
        return feature_hash

class SecureVoiceProcessor:
    def __init__(self):
        self.privacy_manager = VoicePrivacyManager()
        self.retention_policy_seconds = 3600  # 1 hour

    def process_voice_command_securely(self, audio_data: np.ndarray) -> str:
        """Process voice command with privacy and security measures"""
        # Anonymize voice data
        anonymized_audio = self.privacy_manager.anonymize_voice_data(audio_data)

        # Convert to bytes for encryption
        audio_bytes = anonymized_audio.tobytes()

        # Encrypt audio data
        encrypted_audio = self.privacy_manager.encrypt_audio_data(audio_bytes)

        # For Whisper processing, we need to decrypt temporarily
        decrypted_audio = self.privacy_manager.decrypt_audio_data(encrypted_audio)
        processed_audio = np.frombuffer(decrypted_audio, dtype=np.float32)

        # Process with Whisper (this would be the actual Whisper call)
        # For demonstration, we'll return a placeholder
        transcription = "Securely processed voice command"

        # Clean up sensitive data
        del audio_bytes, encrypted_audio, decrypted_audio, processed_audio

        return transcription
```

## Integration with ROS 2 for Command Routing

The voice command system must be properly integrated with ROS 2 for command routing and execution in humanoid robot systems.

```python
import rclpy
from rclpy.node import Node
from std_msgs.msg import String
from sensor_msgs.msg import AudioData
from builtin_interfaces.msg import Duration
import threading
import queue
from typing import Optional

class ROS2VoiceCommandNode(Node):
    def __init__(self):
        super().__init__('ros2_voice_command_node')

        # Publishers
        self.command_publisher = self.create_publisher(String, 'voice_commands', 10)
        self.intent_publisher = self.create_publisher(String, 'voice_intents', 10)
        self.status_publisher = self.create_publisher(String, 'voice_status', 10)

        # Subscribers
        self.audio_subscription = self.create_subscription(
            AudioData,
            'audio_input',
            self.audio_callback,
            10
        )

        # Parameters
        self.declare_parameter('model_size', 'base')
        self.declare_parameter('sample_rate', 16000)
        self.declare_parameter('enable_noise_reduction', True)

        # Get parameters
        self.model_size = self.get_parameter('model_size').value
        self.sample_rate = self.get_parameter('sample_rate').value
        self.enable_noise_reduction = self.get_parameter('enable_noise_reduction').value

        # Initialize Whisper model
        import whisper
        self.model = whisper.load_model(self.model_size)

        # Audio processing queue
        self.audio_queue = queue.Queue()

        # Start audio processing thread
        self.processing_thread = threading.Thread(target=self.process_audio_queue)
        self.processing_thread.daemon = True
        self.processing_thread.start()

        self.get_logger().info('ROS2 Voice Command Node initialized')

    def audio_callback(self, msg: AudioData):
        """Callback for audio input"""
        # Add audio data to processing queue
        self.audio_queue.put(msg.data)

        # Publish status
        status_msg = String()
        status_msg.data = 'audio_received'
        self.status_publisher.publish(status_msg)

    def process_audio_queue(self):
        """Process audio data from the queue"""
        while rclpy.ok():
            try:
                # Get audio data from queue
                audio_data = self.audio_queue.get(timeout=1.0)

                # Convert bytes to numpy array
                import numpy as np
                audio_array = np.frombuffer(audio_data, dtype=np.int16).astype(np.float32) / 32768.0

                # Preprocess audio if enabled
                if self.enable_noise_reduction:
                    # Apply noise reduction (simplified)
                    audio_array = self.apply_noise_reduction(audio_array)

                # Transcribe using Whisper
                result = self.model.transcribe(audio_array)
                transcription = result["text"].strip()

                if transcription:
                    # Publish command
                    command_msg = String()
                    command_msg.data = transcription
                    self.command_publisher.publish(command_msg)

                    # Extract and publish intent
                    intent = self.extract_intent(transcription)
                    intent_msg = String()
                    intent_msg.data = intent
                    self.intent_publisher.publish(intent_msg)

                    self.get_logger().info(f'Transcribed: {transcription}')
                    self.get_logger().info(f'Intent: {intent}')

                    # Publish success status
                    status_msg = String()
                    status_msg.data = 'command_processed'
                    self.status_publisher.publish(status_msg)
            except queue.Empty:
                continue
            except Exception as e:
                self.get_logger().error(f'Error processing audio: {e}')

    def apply_noise_reduction(self, audio_data):
        """Apply simple noise reduction"""
        # Simple noise reduction by spectral gating
        import numpy as np
        from scipy import signal

        # Apply a simple low-pass filter to reduce high-frequency noise
        b, a = signal.butter(4, 0.8, btype='low')
        filtered_audio = signal.filtfilt(b, a, audio_data)

        return filtered_audio

    def extract_intent(self, text: str) -> str:
        """Extract intent from transcribed text"""
        text_lower = text.lower()

        # Intent classification based on keywords
        movement_keywords = ["move", "walk", "go", "forward", "backward", "left", "right", "step"]
        stop_keywords = ["stop", "halt", "pause", "freeze"]
        manipulation_keywords = ["pick", "grab", "take", "lift", "hold", "release"]
        action_keywords = ["dance", "jump", "kick", "wave", "nod"]
        navigation_keywords = ["follow", "come", "here", "to me", "approach"]

        if any(keyword in text_lower for keyword in movement_keywords):
            return "movement_command"
        elif any(keyword in text_lower for keyword in stop_keywords):
            return "stop_command"
        elif any(keyword in text_lower for keyword in manipulation_keywords):
            return "manipulation_command"
        elif any(keyword in text_lower for keyword in action_keywords):
            return "action_command"
        elif any(keyword in text_lower for keyword in navigation_keywords):
            return "navigation_command"
        else:
            return "unknown_command"

def main(args=None):
    rclpy.init(args=args)

    voice_node = ROS2VoiceCommandNode()

    try:
        rclpy.spin(voice_node)
    except KeyboardInterrupt:
        pass
    finally:
        voice_node.destroy_node()
        rclpy.shutdown()

if __name__ == '__main__':
    main()
```

## Whisper Model Optimization for Robotics

Optimizing Whisper for real-time robotics applications requires careful consideration of model size, inference speed, and accuracy trade-offs.

```python
import whisper
import torch
import time
from typing import Dict, Any, Optional
import numpy as np

class OptimizedWhisperProcessor:
    def __init__(self, model_size: str = "base", use_gpu: bool = True):
        self.model_size = model_size
        self.use_gpu = use_gpu and torch.cuda.is_available()

        # Load model with appropriate precision
        if self.use_gpu:
            self.model = whisper.load_model(model_size).cuda()
        else:
            self.model = whisper.load_model(model_size)

        # Warm up the model
        self._warmup()

    def _warmup(self):
        """Warm up the model to ensure optimal performance"""
        # Create a short dummy audio signal
        dummy_audio = np.random.randn(16000).astype(np.float32)

        try:
            # Run a dummy transcription to warm up the model
            result = self.model.transcribe(dummy_audio, fp16=self.use_gpu)
        except Exception as e:
            print(f"Warmup failed: {e}")

    def transcribe_optimized(self, audio_data: np.ndarray, language: str = "en") -> Dict[str, Any]:
        """Optimized transcription with performance monitoring"""
        start_time = time.time()

        # Ensure audio is in correct format
        if len(audio_data.shape) > 1:
            audio_data = audio_data.squeeze()

        # Transcribe using Whisper with optimizations
        options = {
            "language": language,
            "fp16": self.use_gpu,  # Use float16 on GPU for speed
            "without_timestamps": True,
            "temperature": [0.0, 0.2, 0.4, 0.6, 0.8, 1.0]  # Multiple temperatures for robustness
        }

        result = self.model.transcribe(audio_data, **options)

        end_time = time.time()
        processing_time = end_time - start_time

        # Add performance metrics to result
        result["processing_time"] = processing_time
        result["model_size"] = self.model_size
        result["used_gpu"] = self.use_gpu

        return result

    def batch_transcribe(self, audio_segments: list) -> list:
        """Process multiple audio segments efficiently"""
        results = []

        for audio_segment in audio_segments:
            result = self.transcribe_optimized(audio_segment)
            results.append(result)

        return results

    def get_model_info(self) -> Dict[str, Any]:
        """Get information about the loaded model"""
        return {
            "model_size": self.model_size,
            "use_gpu": self.use_gpu,
            "device": "cuda" if self.use_gpu else "cpu",
            "model_type": self.model.model.dims,
            "parameters": sum(p.numel() for p in self.model.parameters())
        }

class AdaptiveWhisperSelector:
    def __init__(self):
        self.model_cache = {}
        self.performance_stats = {}

    def select_model(self, audio_duration: float, required_accuracy: float = 0.8) -> str:
        """Select appropriate Whisper model based on requirements"""
        if audio_duration < 5:  # Short audio
            if required_accuracy > 0.9:
                return "large"  # High accuracy required
            else:
                return "base"   # Medium accuracy sufficient
        elif audio_duration < 30:  # Medium audio
            if required_accuracy > 0.9:
                return "large"
            elif required_accuracy > 0.8:
                return "medium"
            else:
                return "base"
        else:  # Long audio
            return "medium"  # Balance between accuracy and speed

    def get_optimized_processor(self, audio_duration: float, required_accuracy: float = 0.8) -> OptimizedWhisperProcessor:
        """Get optimized processor for specific requirements"""
        model_size = self.select_model(audio_duration, required_accuracy)

        if model_size not in self.model_cache:
            self.model_cache[model_size] = OptimizedWhisperProcessor(model_size)

        return self.model_cache[model_size]
```

## Summary

This lesson provided a comprehensive overview of voice-to-text intake using Whisper in humanoid robot systems. We explored advanced Whisper integration with audio preprocessing, noise reduction, and voice activity detection. The implementation includes privacy and security considerations, ROS 2 integration for command routing, and optimization techniques for real-time robotics applications. The advanced architecture handles multi-user environments, speaker identification, and provides robust voice command processing capabilities essential for effective human-robot interaction.

## References

1. Radford, A., Kim, J. W., Xu, T., Khabsa, G., Goyal, N., & Mann, T. (2022). Robust speech recognition via large-scale weak supervision. *arXiv preprint arXiv:2212.04356*.

2. Higuchi, T., Ito, N., Araki, S., Yoshioka, T., & Nakatani, T. (2017). Robust MVDR beamforming using time-frequency masks for online/offline ASR in noise. *2017 IEEE International Conference on Acoustics, Speech and Signal Processing (ICASSP)*, 5210-5214.

3. Panayotov, V., Chen, G., Povey, D., & Khudanpur, S. (2015). LibriSpeech: An ASR corpus based on public domain audio books. *2015 IEEE International Conference on Acoustics, Speech and Signal Processing (ICASSP)*, 5206-5210.

4. Grais, E. M., & Plumbley, M. D. (2017). Single channel audio source separation using convolutional denoising autoencoders. *arXiv preprint arXiv:1703.06205*.

5. Yin, Z., Wang, Y., Liu, X., & Liu, Y. (2020). Voice privacy challenge 2020: results and findings. *arXiv preprint arXiv:2102.01167*.

6. Vasquez-Leal, P., & Bello, R. P. (2021). Voice-based human-robot interaction: A survey. *IEEE Access*, 9, 125434-125453.