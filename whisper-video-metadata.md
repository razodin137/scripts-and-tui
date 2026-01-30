import os
import time
from datetime import datetime
from moviepy import VideoFileClip
from mutagen.mp4 import MP4, MP4Cover
from dotenv import load_dotenv
from openai import OpenAI
from pydub import AudioSegment
import math
import re
import json
import cv2
import numpy as np
import speech_recognition as sr
from sklearn.cluster import KMeans
import librosa
import sqlite3
from pathlib import Path

def retry_with_backoff(func, max_retries=3, initial_delay=1):
    """Retry a function with exponential backoff."""
    def wrapper(*args, **kwargs):
        delay = initial_delay
        last_exception = None
        
        for retry in range(max_retries):
            try:
                return func(*args, **kwargs)
            except Exception as e:
                last_exception = e
                if retry < max_retries - 1:
                    print(f"├── Attempt {retry + 1} failed: {str(e)}")
                    print(f"├── Retrying in {delay} seconds...")
                    time.sleep(delay)
                    delay *= 2
        
        print(f"├── All {max_retries} attempts failed")
        raise last_exception
    
    return wrapper

# Load API key from .env file
load_dotenv()

# Initialize the client (after load_dotenv())
client = OpenAI()

# Folder paths
INPUT_FOLDER = "/home/mrjohn/Desktop/whisper-metadata-adder/input"
PROCESSED_FOLDER = "/home/mrjohn/Desktop/whisper-metadata-adder/processed"
FAILED_FOLDER = "/home/mrjohn/Desktop/whisper-metadata-adder/failed"

# Ensure folders exist
os.makedirs(INPUT_FOLDER, exist_ok=True)
os.makedirs(PROCESSED_FOLDER, exist_ok=True)
os.makedirs(FAILED_FOLDER, exist_ok=True)

# Function to retry API calls with exponential backoff
@retry_with_backoff
def api_transcribe(file, response_format="text"):
    """Wrapper for OpenAI API call with retries."""
    return client.audio.transcriptions.create(
        model="whisper-1",
        file=file,
        response_format=response_format,
        timestamp_granularities=["word"] if response_format == "srt" else None
    )

# Function to retry chat completion API calls with exponential backoff
@retry_with_backoff
def api_chat_completion(messages, model="gpt-4", temperature=0.7):
    """Wrapper for OpenAI chat completion API call with retries."""
    return client.chat.completions.create(
        model=model,
        messages=messages,
        temperature=temperature
    )

# Function to transcribe audio using Whisper API
def transcribe_audio(file_path, response_format="text"):
    temp_audio_path = None
    try:
        if file_path.endswith(('.mp4', '.mov', '.mkv')):
            # Check file size before processing
            file_size = os.path.getsize(file_path)
            max_file_size = 2 * 1024 * 1024 * 1024  # 2GB limit
            if file_size > max_file_size:
                raise ValueError(f"File too large: {file_size/1024/1024:.2f}MB exceeds limit of {max_file_size/1024/1024}MB")
                
            print("├── Converting video to audio...")
            video = VideoFileClip(file_path)
            # Create temporary audio file with compression
            temp_audio_path = file_path + '.temp.wav'
            
            try:
                print("├── Compressing audio to meet API limits...")
                video.audio.write_audiofile(
                    temp_audio_path,
                    fps=16000,  # Lower sample rate
                    nbytes=2,   # 16-bit audio
                    codec='pcm_s16le'  # Use 16-bit codec
                )
            finally:
                video.close()
            
            # Check file size
            file_size = os.path.getsize(temp_audio_path)
            max_size = 25 * 1024 * 1024  # 25MB in bytes
            
            if file_size > max_size:
                print(f"├── Audio file size: {file_size/1024/1024:.2f}MB")
                print("├── Splitting audio into chunks...")
                
                # Load audio file
                audio = AudioSegment.from_wav(temp_audio_path)
                duration_ms = len(audio)
                
                # Calculate number of chunks needed (with 20% margin for safety)
                chunk_size_ms = int((duration_ms / (file_size / max_size)) * 0.8)
                num_chunks = math.ceil(duration_ms / chunk_size_ms)
                
                print(f"├── Splitting into {num_chunks} chunks...")
                
                # Process each chunk
                transcription_parts = []
                for i in range(num_chunks):
                    start_ms = i * chunk_size_ms
                    end_ms = min((i + 1) * chunk_size_ms, duration_ms)
                    
                    chunk = audio[start_ms:end_ms]
                    chunk_path = f"{temp_audio_path}.chunk{i}.wav"
                    chunk.export(chunk_path, format="wav")
                    
                    print(f"├── Processing chunk {i+1}/{num_chunks}...")
                    with open(chunk_path, "rb") as audio_file:
                        chunk_response = api_transcribe(audio_file, response_format)
                        transcription_parts.append(chunk_response)
                    os.remove(chunk_path)
                
                # Clean up temporary file
                os.remove(temp_audio_path)
                if response_format == "srt":
                    # Adjust timestamps for each chunk
                    adjusted_srt = []
                    for i, chunk_srt in enumerate(transcription_parts):
                        chunk_start = i * chunk_size_ms / 1000  # Convert to seconds
                        adjusted_chunk = adjust_srt_timestamps(chunk_srt, chunk_start)
                        adjusted_srt.append(adjusted_chunk)
                    return "\n".join(adjusted_srt)
                else:
                    return " ".join(transcription_parts)
            
            print("├── Sending to OpenAI Whisper API...")
            with open(temp_audio_path, "rb") as audio_file:
                print("├── File opened, making API request...")
                response = api_transcribe(audio_file, response_format)
                print(f"├── Received response from API")
            
            # Clean up temporary file
            print("├── Cleaning up temporary files...")
            os.remove(temp_audio_path)
            return response
        else:
            # Direct transcription for audio files
            with open(file_path, "rb") as audio_file:
                response = api_transcribe(audio_file, response_format)
                return response
    except Exception as e:
        print(f"Error transcribing audio: {e}")
        print(f"├── Error details: {str(e.__class__.__name__)}")
        if os.path.exists(temp_audio_path):
            os.remove(temp_audio_path)
        return None

def adjust_srt_timestamps(srt_content, offset_seconds):
    """Adjust SRT timestamps by adding an offset in seconds"""
    lines = srt_content.split('\n')
    adjusted_lines = []
    
    for line in lines:
        if '-->' in line:
            start, end = line.split(' --> ')
            adjusted_start = add_seconds_to_timestamp(start.strip(), offset_seconds)
            adjusted_end = add_seconds_to_timestamp(end.strip(), offset_seconds)
            adjusted_lines.append(f"{adjusted_start} --> {adjusted_end}")
        else:
            adjusted_lines.append(line)
    
    return '\n'.join(adjusted_lines)

def add_seconds_to_timestamp(timestamp, seconds):
    """Add seconds to an SRT timestamp"""
    h, m, s = timestamp.replace(',', '.').split(':')
    total_seconds = float(h) * 3600 + float(m) * 60 + float(s) + seconds
    h = int(total_seconds // 3600)
    m = int((total_seconds % 3600) // 60)
    s = total_seconds % 60
    return f"{h:02d}:{m:02d}:{s:06.3f}".replace('.', ',')

def analyze_video_content(file_path):
    """Analyze video content for enhanced metadata"""
    video = VideoFileClip(file_path)
    cap = cv2.VideoCapture(file_path)
    
    # Basic video metrics
    metrics = {
        "duration": video.duration,
        "resolution": f"{video.size[0]}x{video.size[1]}",
        "fps": video.fps,
        "audio_channels": video.audio.nchannels if video.audio else 0,
    }
    
    # Scene detection
    scenes = []
    prev_frame = None
    frame_count = 0
    while cap.isOpened():
        ret, frame = cap.read()
        if not ret:
            break
            
        if prev_frame is not None:
            # Simple scene detection using frame difference
            diff = np.mean(np.abs(frame - prev_frame))
            if diff > 30:  # Threshold for scene change
                timestamp = frame_count / video.fps
                scenes.append(timestamp)
        
        prev_frame = frame
        frame_count += 1
        
        # Sample every 5 frames for performance
        for _ in range(4):
            cap.read()
            frame_count += 1
    
    metrics["scene_changes"] = scenes
    
    # Audio analysis
    if video.audio:
        audio = video.audio
        audio_array = audio.to_soundarray()
        
        # Audio quality metrics
        metrics["audio_quality"] = {
            "sample_rate": audio.fps,
            "rms_energy": float(np.sqrt(np.mean(audio_array**2))),
            "peak_amplitude": float(np.max(np.abs(audio_array))),
        }
        
        # Rough speaker diarization using audio segments
        audio_segments = librosa.effects.split(
            audio_array.mean(axis=1) if len(audio_array.shape) > 1 else audio_array,
            top_db=20
        )
        
        # Cluster audio segments for potential different speakers
        if len(audio_segments) > 0:
            segment_features = []
            for start, end in audio_segments:
                segment = audio_array[start:end]
                mfccs = librosa.feature.mfcc(y=segment.mean(axis=1) if len(segment.shape) > 1 else segment, 
                                           sr=int(audio.fps))
                segment_features.append(np.mean(mfccs, axis=1))
            
            if segment_features:
                n_speakers = min(len(segment_features), 3)  # Assume max 3 speakers
                kmeans = KMeans(n_clusters=n_speakers)
                speaker_labels = kmeans.fit_predict(segment_features)
                
                speakers = []
                for i, (start, end) in enumerate(audio_segments):
                    speakers.append({
                        "start": float(start / audio.fps),
                        "end": float(end / audio.fps),
                        "speaker_id": int(speaker_labels[i])
                    })
                
                metrics["speaker_segments"] = speakers
    
    cap.release()
    video.close()
    
    return metrics

def create_semantic_metadata(metadata_json, transcription, video_metrics):
    """Create structured semantic metadata using JSON-LD schema"""
    
    # Split transcript into topics using simple heuristics
    sentences = transcription.split('. ')
    topics = []
    current_topic = []
    current_topic_keywords = set()
    
    for sentence in sentences:
        # Extract keywords from sentence
        words = set(word.lower() for word in sentence.split() 
                   if len(word) > 4 and word.isalnum())
        
        # If significant keyword overlap, continue topic
        if len(current_topic_keywords & words) >= 2:
            current_topic.append(sentence)
            current_topic_keywords.update(words)
        else:
            # Save previous topic if it exists
            if current_topic:
                topics.append({
                    "content": '. '.join(current_topic),
                    "keywords": list(current_topic_keywords)
                })
            # Start new topic
            current_topic = [sentence]
            current_topic_keywords = words
    
    # Add final topic
    if current_topic:
        topics.append({
            "content": '. '.join(current_topic),
            "keywords": list(current_topic_keywords)
        })
    
    # Create JSON-LD structure
    semantic_metadata = {
        "@context": "https://schema.org",
        "@type": "VideoObject",
        "name": metadata_json['title'],
        "description": metadata_json['description'],
        "duration": f"PT{int(video_metrics['duration'])}S",
        "thumbnailUrl": "",  # Could be generated from scene detection
        "uploadDate": datetime.now().isoformat(),
        "transcript": {
            "text": transcription,
            "topics": topics
        },
        "contentDetails": {
            "resolution": video_metrics['resolution'],
            "frameRate": video_metrics['fps'],
            "scenes": [{"timestamp": t} for t in video_metrics.get('scene_changes', [])],
            "audioQuality": video_metrics.get('audio_quality', {}),
            "speakers": video_metrics.get('speaker_segments', [])
        },
        "keywords": metadata_json['hashtags']
    }
    
    return semantic_metadata

def add_metadata(file_path, metadata_json, transcription):
    try:
        video = MP4(file_path)
        
        # Parse metadata_json if it's a string
        if isinstance(metadata_json, str):
            try:
                metadata_json = json.loads(metadata_json)
            except json.JSONDecodeError as e:
                print(f"Error parsing metadata JSON: {e}")
                raise
        
        print("├── Analyzing video content...")
        video_metrics = analyze_video_content(file_path)
        
        print("├── Creating semantic metadata...")
        semantic_metadata = create_semantic_metadata(metadata_json, transcription, video_metrics)
        
        # Standard iTunes/MP4 metadata tags
        video["\xa9nam"] = metadata_json['title']  # Title
        video["\xa9ART"] = "Generated by Whisper"  # Artist
        video["\xa9alb"] = "Whisper Transcription"  # Album
        video["\xa9day"] = datetime.now().strftime("%Y")  # Year
        video["\xa9des"] = metadata_json['description']  # Description
        video["\xa9cmt"] = metadata_json['summary']  # Comment
        video["keyw"] = ", ".join(metadata_json['hashtags'])  # Keywords
        video["desc"] = metadata_json['description']  # Long Description
        
        # Store full semantic metadata as JSON
        video["----:com.whisper.metadata:SEMANTIC"] = bytes(json.dumps(semantic_metadata), 'utf-8')
        
        # Add chapter markers based on detected scenes
        if video_metrics.get('scene_changes'):
            chapter_titles = [f"Scene {i+1}" for i in range(len(video_metrics['scene_changes']))]
            video["chpl"] = create_chapter_list(video_metrics['scene_changes'], chapter_titles)
        
        # Additional metadata tags with proper encoding
        for key, value in {
            "©gen": "Whisper Transcription",  # Genre
            "©too": "Whisper Video Metadata",  # Encoding Tool
            "©wrt": transcription[:2000],  # Transcript (first 2000 chars)
            "©lyr": transcription[:2000],  # Lyrics field for transcript
            "©tag": ", ".join(metadata_json['hashtags']),  # Tags
            "©cpy": f"Generated {datetime.now().strftime('%Y-%m-%d')}",  # Copyright
        }.items():
            try:
                if isinstance(value, str):
                    video[key] = value
            except Exception as e:
                print(f"Warning: Could not set metadata key {key}: {e}")
        
        # Save changes
        video.save()
        print("├── Successfully added enhanced metadata to video file")
        
    except Exception as e:
        print(f"Error inserting metadata: {e}")
        raise

def validate_metadata_json(metadata):
    """Validate the structure of metadata JSON and provide defaults if needed."""
    required_keys = ['title', 'summary', 'description', 'hashtags']
    
    if not isinstance(metadata, dict):
        raise ValueError("Metadata must be a dictionary")
    
    # Ensure all required keys exist
    for key in required_keys:
        if key not in metadata:
            if key == 'hashtags':
                metadata[key] = []  # Default empty list for hashtags
            else:
                metadata[key] = "No " + key  # Default text for missing fields
    
    # Ensure hashtags is a list
    if not isinstance(metadata['hashtags'], list):
        # If it's a string, try to split it
        if isinstance(metadata['hashtags'], str):
            metadata['hashtags'] = metadata['hashtags'].split()
        else:
            metadata['hashtags'] = []
    
    return metadata

def generate_content_metadata(transcription):
    print("├── Generating content metadata...")
    try:
        response = api_chat_completion(
            messages=[
                {"role": "system", "content": """Generate comprehensive metadata for a video based on its transcript. Return a JSON object with:
                {
                    "title": "A concise, engaging title (max 60 chars)",
                    "summary": "A compelling summary (2-3 sentences) suitable for social media",
                    "description": "A detailed description (2-3 paragraphs) that thoroughly explains the content",
                    "hashtags": []  # Array of relevant hashtags
                }
                
                Ensure the response is valid JSON and hashtags are provided as an array."""},
                {"role": "user", "content": transcription[:4000]}  # First 4000 chars for context
            ]
        )
        
        try:
            # Parse the response content as JSON
            metadata = json.loads(response.choices[0].message.content)
            # Validate and fix the metadata structure
            metadata = validate_metadata_json(metadata)
            return metadata
        except json.JSONDecodeError as e:
            print(f"├── Error parsing JSON response: {str(e)}")
            # Try to extract JSON from the response if it contains additional text
            match = re.search(r'\{.*\}', response.choices[0].message.content, re.DOTALL)
            if match:
                try:
                    metadata = json.loads(match.group(0))
                    metadata = validate_metadata_json(metadata)
                    return metadata
                except:
                    pass
            return None
    except Exception as e:
        print(f"├── Error generating metadata: {str(e)}")
        return None

def create_caption_files(file_path, transcription, metadata_json):
    """Create caption files with proper error handling and atomic writes."""
    temp_files = []
    try:
        base_name = os.path.splitext(file_path)[0]
        
        # Create markdown file with structured format
        md_path = f"{base_name}.captions.md"
        temp_md_path = md_path + '.tmp'
        temp_files.append(temp_md_path)
        
        print(f"├── Creating markdown file: {md_path}")
        with open(temp_md_path, 'w', encoding='utf-8') as f:
            try:
                # Title and metadata section
                f.write(f"# {metadata_json['title']}\n\n")
                
                # Executive Summary
                f.write("## Executive Summary\n")
                f.write(f"{metadata_json['summary']}\n\n")
                
                # Social Media Section
                f.write("## Social Media Content\n")
                f.write("### Caption\n")
                f.write(f"{metadata_json['summary']}\n\n")
                hashtags = ' '.join([f'#{tag}' for tag in metadata_json['hashtags']])
                f.write(f"### Hashtags\n{hashtags}\n\n")
                
                # Detailed Description
                f.write("## Detailed Description\n")
                f.write(f"{metadata_json['description']}\n\n")
                
                # Full Transcript formatted as an essay
                f.write("## Full Transcript\n")
                paragraphs = format_transcript_as_essay(transcription)
                f.write(paragraphs + "\n")
            except KeyError as e:
                print(f"├── Warning: Missing metadata field {str(e)}")
                # Continue with available data
        
        # Atomic rename of markdown file
        os.replace(temp_md_path, md_path)
        temp_files.remove(temp_md_path)

        # Create SRT file with accurate timestamps
        srt_path = f"{base_name}.srt"
        temp_srt_path = srt_path + '.tmp'
        temp_files.append(temp_srt_path)
        
        print(f"├── Creating SRT file: {srt_path}")
        srt_content = transcribe_audio(file_path, response_format="srt")
        if srt_content:
            with open(temp_srt_path, 'w', encoding='utf-8') as srt_file:
                srt_file.write(srt_content)
            # Atomic rename of SRT file
            os.replace(temp_srt_path, srt_path)
            temp_files.remove(temp_srt_path)

    except Exception as e:
        print(f"├── Error in create_caption_files: {str(e)}")
        raise
    finally:
        # Cleanup any remaining temporary files
        for temp_file in temp_files:
            try:
                if os.path.exists(temp_file):
                    os.remove(temp_file)
            except Exception as e:
                print(f"├── Warning: Failed to clean up temporary file {temp_file}: {str(e)}")

def format_transcript_as_essay(transcription):
    """Format the transcript into proper paragraphs"""
    # Split into sentences
    sentences = re.split(r'(?<=[.!?])\s+', transcription)
    
    # Group sentences into paragraphs (roughly 4-5 sentences per paragraph)
    paragraphs = []
    current_paragraph = []
    
    for sentence in sentences:
        current_paragraph.append(sentence)
        if len(current_paragraph) >= 4 and sentence.endswith(('.', '!', '?')):
            paragraphs.append(' '.join(current_paragraph))
            current_paragraph = []
    
    # Add any remaining sentences
    if current_paragraph:
        paragraphs.append(' '.join(current_paragraph))
    
    # Join paragraphs with double newlines
    return '\n\n'.join(paragraphs)

def process_large_file(file_path, chunk_size_mb=25):
    """Process large files in chunks to manage memory usage."""
    temp_audio_path = None
    try:
        # Convert video to audio
        video = VideoFileClip(file_path)
        temp_audio_path = file_path + '.temp.wav'
        
        try:
            video.audio.write_audiofile(
                temp_audio_path,
                fps=16000,
                nbytes=2,
                codec='pcm_s16le'
            )
        finally:
            video.close()
        
        # Get audio file size
        file_size = os.path.getsize(temp_audio_path)
        chunk_size_bytes = chunk_size_mb * 1024 * 1024
        
        if file_size <= chunk_size_bytes:
            # Process small file directly
            with open(temp_audio_path, "rb") as audio_file:
                return api_transcribe(audio_file)
        
        # Process large file in chunks
        print(f"├── Processing large file in chunks ({file_size/1024/1024:.2f}MB)")
        audio = AudioSegment.from_wav(temp_audio_path)
        duration_ms = len(audio)
        
        # Calculate chunk size in milliseconds
        chunk_duration = int((chunk_size_bytes / file_size) * duration_ms)
        num_chunks = math.ceil(duration_ms / chunk_duration)
        
        transcription_parts = []
        for i in range(num_chunks):
            start_ms = i * chunk_duration
            end_ms = min((i + 1) * chunk_duration, duration_ms)
            
            print(f"├── Processing chunk {i+1}/{num_chunks}")
            chunk = audio[start_ms:end_ms]
            chunk_path = f"{temp_audio_path}.chunk{i}.wav"
            
            try:
                chunk.export(chunk_path, format="wav")
                with open(chunk_path, "rb") as chunk_file:
                    result = api_transcribe(chunk_file)
                    transcription_parts.append(result)
            finally:
                if os.path.exists(chunk_path):
                    os.remove(chunk_path)
        
        return " ".join(transcription_parts)
        
    finally:
        if temp_audio_path and os.path.exists(temp_audio_path):
            os.remove(temp_audio_path)

def init_database():
    """Initialize SQLite database for video metadata"""
    db_path = Path(os.path.dirname(INPUT_FOLDER)) / "video_metadata.db"
    conn = sqlite3.connect(str(db_path))
    c = conn.cursor()
    
    # Create tables
    c.execute('''CREATE TABLE IF NOT EXISTS videos
                 (id INTEGER PRIMARY KEY AUTOINCREMENT,
                  filename TEXT UNIQUE,
                  title TEXT,
                  description TEXT,
                  duration REAL,
                  resolution TEXT,
                  fps REAL,
                  process_date TIMESTAMP,
                  semantic_metadata TEXT)''')
    
    c.execute('''CREATE TABLE IF NOT EXISTS transcripts
                 (id INTEGER PRIMARY KEY AUTOINCREMENT,
                  video_id INTEGER,
                  full_text TEXT,
                  FOREIGN KEY(video_id) REFERENCES videos(id))''')
    
    c.execute('''CREATE TABLE IF NOT EXISTS topics
                 (id INTEGER PRIMARY KEY AUTOINCREMENT,
                  video_id INTEGER,
                  content TEXT,
                  keywords TEXT,
                  start_time REAL,
                  end_time REAL,
                  FOREIGN KEY(video_id) REFERENCES videos(id))''')
    
    c.execute('''CREATE TABLE IF NOT EXISTS speakers
                 (id INTEGER PRIMARY KEY AUTOINCREMENT,
                  video_id INTEGER,
                  speaker_id INTEGER,
                  start_time REAL,
                  end_time REAL,
                  FOREIGN KEY(video_id) REFERENCES videos(id))''')
    
    c.execute('''CREATE TABLE IF NOT EXISTS scenes
                 (id INTEGER PRIMARY KEY AUTOINCREMENT,
                  video_id INTEGER,
                  timestamp REAL,
                  title TEXT,
                  FOREIGN KEY(video_id) REFERENCES videos(id))''')
    
    # Create full-text search index
    c.execute('''CREATE VIRTUAL TABLE IF NOT EXISTS transcript_search 
                 USING fts5(video_id, content)''')
    
    conn.commit()
    return conn

def store_video_metadata(conn, file_path, metadata_json, semantic_metadata, transcription):
    """Store video metadata in SQLite database"""
    c = conn.cursor()
    
    # Store basic video info
    c.execute('''INSERT INTO videos 
                 (filename, title, description, duration, resolution, fps, 
                  process_date, semantic_metadata)
                 VALUES (?, ?, ?, ?, ?, ?, ?, ?)''',
              (os.path.basename(file_path),
               metadata_json['title'],
               metadata_json['description'],
               semantic_metadata['contentDetails']['duration'],
               semantic_metadata['contentDetails']['resolution'],
               semantic_metadata['contentDetails']['frameRate'],
               datetime.now(),
               json.dumps(semantic_metadata)))
    
    video_id = c.lastrowid
    
    # Store transcript
    c.execute('INSERT INTO transcripts (video_id, full_text) VALUES (?, ?)',
              (video_id, transcription))
    
    # Store in search index
    c.execute('INSERT INTO transcript_search (video_id, content) VALUES (?, ?)',
              (video_id, transcription))
    
    # Store topics
    for topic in semantic_metadata['transcript']['topics']:
        c.execute('''INSERT INTO topics 
                     (video_id, content, keywords)
                     VALUES (?, ?, ?)''',
                  (video_id,
                   topic['content'],
                   json.dumps(topic['keywords'])))
    
    # Store speaker segments
    for speaker in semantic_metadata['contentDetails']['speakers']:
        c.execute('''INSERT INTO speakers 
                     (video_id, speaker_id, start_time, end_time)
                     VALUES (?, ?, ?, ?)''',
                  (video_id,
                   speaker['speaker_id'],
                   speaker['start'],
                   speaker['end']))
    
    # Store scenes
    for scene in semantic_metadata['contentDetails']['scenes']:
        c.execute('''INSERT INTO scenes 
                     (video_id, timestamp, title)
                     VALUES (?, ?, ?)''',
                  (video_id,
                   scene['timestamp'],
                   f"Scene at {scene['timestamp']:.2f}s"))
    
    conn.commit()

def process_files():
    # Initialize database
    conn = init_database()
    
    while True:
        print("\n=== Whisper Video Metadata Processor Started ===")
        print(f"Watching folder: {INPUT_FOLDER}")
        print("Press Ctrl+C to stop\n")

        files = [f for f in os.listdir(INPUT_FOLDER) if f.endswith((".mp4", ".m4a", ".mov", ".mkv"))]
        if not files:
            print(f"[{datetime.now()}] No files found. Waiting...")
            time.sleep(5)
            continue

        print(f"[{datetime.now()}] Found {len(files)} file(s).")
        
        for file_name in files:
            file_path = os.path.join(INPUT_FOLDER, file_name)
            try:
                if not os.path.exists(file_path):
                    print(f"File no longer exists: {file_path}")
                    continue
                    
                print(f"\nProcessing: {file_name}")
                
                # Check file size before processing
                file_size = os.path.getsize(file_path)
                max_file_size = 2 * 1024 * 1024 * 1024  # 2GB limit
                if file_size > max_file_size:
                    raise ValueError(f"File too large: {file_size/1024/1024:.2f}MB exceeds limit of {max_file_size/1024/1024}MB")
                
                print("├── Transcribing audio...")
                transcription = process_large_file(file_path)
                if transcription is None:
                    raise Exception("Transcription failed.")

                print("├── Generating content metadata...")
                metadata_json = generate_content_metadata(transcription)
                if metadata_json is None:
                    raise Exception("Metadata generation failed.")

                print("├── Adding metadata to video file...")
                video_metrics = analyze_video_content(file_path)
                semantic_metadata = create_semantic_metadata(metadata_json, transcription, video_metrics)
                add_metadata(file_path, metadata_json, transcription)
                
                print("├── Storing metadata in database...")
                store_video_metadata(conn, file_path, metadata_json, semantic_metadata, transcription)

                print("├── Creating caption files...")
                create_caption_files(file_path, transcription, metadata_json)

                print("├── Moving files to processed folder...")
                base_name = os.path.splitext(file_name)[0]
                for ext in ['.srt', '.captions.md']:
                    source_path = os.path.join(INPUT_FOLDER, f"{base_name}{ext}")
                    if os.path.exists(source_path):
                        dest_path = os.path.join(PROCESSED_FOLDER, f"{base_name}{ext}")
                        print(f"├── Moving {source_path} to {dest_path}")
                        os.replace(source_path, dest_path)
                    else:
                        print(f"├── Warning: {source_path} not found")

                # Move the video file
                processed_path = os.path.join(PROCESSED_FOLDER, file_name)
                if os.path.exists(file_path):
                    os.replace(file_path, processed_path)
                    print(f"└── ✓ Successfully processed: {file_name}")

            except Exception as e:
                print(f"└── ✗ Failed to process {file_name}")
                print(f"    Error: {e}")
                failed_path = os.path.join(FAILED_FOLDER, file_name)
                if os.path.exists(file_path):  # Check if file still exists before moving
                    os.replace(file_path, failed_path)

        time.sleep(5)

def search_transcripts(query, conn, limit=10):
    """Search through video transcripts using FTS5"""
    c = conn.cursor()
    c.execute('''SELECT v.filename, v.title, ts.content, ts.rank
                 FROM transcript_search ts
                 JOIN videos v ON ts.video_id = v.id
                 WHERE transcript_search MATCH ?
                 ORDER BY ts.rank
                 LIMIT ?''',
              (query, limit))
    return c.fetchall()

def search_by_topic(topic_keywords, conn, limit=10):
    """Search videos by topic keywords"""
    c = conn.cursor()
    keyword_pattern = f"%{topic_keywords}%"
    c.execute('''SELECT v.filename, v.title, t.content, t.keywords
                 FROM topics t
                 JOIN videos v ON t.video_id = v.id
                 WHERE t.keywords LIKE ?
                 LIMIT ?''',
              (keyword_pattern, limit))
    return c.fetchall()

def get_speaker_segments(video_id, conn):
    """Get all speaker segments for a video"""
    c = conn.cursor()
    c.execute('''SELECT speaker_id, start_time, end_time
                 FROM speakers
                 WHERE video_id = ?
                 ORDER BY start_time''',
              (video_id,))
    return c.fetchall()

def get_scene_markers(video_id, conn):
    """Get all scene markers for a video"""
    c = conn.cursor()
    c.execute('''SELECT timestamp, title
                 FROM scenes
                 WHERE video_id = ?
                 ORDER BY timestamp''',
              (video_id,))
    return c.fetchall()

def get_video_details(video_id, conn):
    """Get comprehensive video details including metadata"""
    c = conn.cursor()
    
    # Get basic video info
    c.execute('SELECT * FROM videos WHERE id = ?', (video_id,))
    video_info = c.fetchone()
    
    if not video_info:
        return None
    
    # Get transcript
    c.execute('SELECT full_text FROM transcripts WHERE video_id = ?', (video_id,))
    transcript = c.fetchone()
    
    # Get topics
    c.execute('SELECT content, keywords FROM topics WHERE video_id = ?', (video_id,))
    topics = c.fetchall()
    
    # Get speaker segments
    c.execute('''SELECT speaker_id, start_time, end_time 
                 FROM speakers 
                 WHERE video_id = ? 
                 ORDER BY start_time''', (video_id,))
    speakers = c.fetchall()
    
    # Get scenes
    c.execute('''SELECT timestamp, title 
                 FROM scenes 
                 WHERE video_id = ? 
                 ORDER BY timestamp''', (video_id,))
    scenes = c.fetchall()
    
    return {
        'video_info': video_info,
        'transcript': transcript[0] if transcript else None,
        'topics': topics,
        'speakers': speakers,
        'scenes': scenes
    }

def export_video_metadata(video_id, conn, output_format='json'):
    """Export video metadata in various formats"""
    details = get_video_details(video_id, conn)
    if not details:
        return None
        
    if output_format == 'json':
        return json.dumps(details, indent=2)
    elif output_format == 'md':
        md_content = f"# {details['video_info'][2]}\n\n"  # Title
        md_content += f"## Description\n{details['video_info'][3]}\n\n"  # Description
        md_content += "## Topics\n"
        for topic in details['topics']:
            md_content += f"- {topic[0]}\n"
        md_content += "\n## Transcript\n"
        md_content += details['transcript']
        return md_content
    else:
        raise ValueError(f"Unsupported output format: {output_format}")

def find_related_videos(video_id, conn, limit=5):
    """Find related videos based on topic similarity"""
    c = conn.cursor()
    
    # Get topics for the source video
    c.execute('SELECT keywords FROM topics WHERE video_id = ?', (video_id,))
    source_topics = c.fetchall()
    
    if not source_topics:
        return []
    
    # Create a search pattern from keywords
    keywords = set()
    for topic in source_topics:
        keywords.update(json.loads(topic[0]))
    
    # Find videos with similar topics
    related_videos = []
    for keyword in keywords:
        c.execute('''SELECT DISTINCT v.id, v.filename, v.title, COUNT(*) as matches
                     FROM videos v
                     JOIN topics t ON v.id = t.video_id
                     WHERE t.keywords LIKE ? AND v.id != ?
                     GROUP BY v.id
                     ORDER BY matches DESC
                     LIMIT ?''',
                  (f"%{keyword}%", video_id, limit))
        related_videos.extend(c.fetchall())
    
    # Sort by number of matching topics and remove duplicates
    seen = set()
    unique_related = []
    for video in sorted(related_videos, key=lambda x: x[3], reverse=True):
        if video[0] not in seen:
            seen.add(video[0])
            unique_related.append(video)
    
    return unique_related[:limit]

if __name__ == "__main__":
    try:
        process_files()
    except KeyboardInterrupt:
        print("\nStopping the processor. Goodbye!")

