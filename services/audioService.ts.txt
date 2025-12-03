import { Track, TrackGenre } from "../types";

// Helper to convert AudioBuffer to WAV Blob for download
const bufferToWave = (abuffer: AudioBuffer, len: number) => {
  let numOfChan = abuffer.numberOfChannels,
      length = len * numOfChan * 2 + 44,
      buffer = new ArrayBuffer(length),
      view = new DataView(buffer),
      channels = [], i, sample,
      offset = 0,
      pos = 0;

  // write WAVE header
  setUint32(0x46464952);                         // "RIFF"
  setUint32(length - 8);                         // file length - 8
  setUint32(0x45564157);                         // "WAVE"

  setUint32(0x20746d66);                         // "fmt " chunk
  setUint32(16);                                 // length = 16
  setUint16(1);                                  // PCM (uncompressed)
  setUint16(numOfChan);
  setUint32(abuffer.sampleRate);
  setUint32(abuffer.sampleRate * 2 * numOfChan); // avg. bytes/sec
  setUint16(numOfChan * 2);                      // block-align
  setUint16(16);                                 // 16-bit (hardcoded in this parser)

  setUint32(0x61746164);                         // "data" - chunk
  setUint32(length - pos - 4);                   // chunk length

  // write interleaved data
  for(i = 0; i < abuffer.numberOfChannels; i++)
    channels.push(abuffer.getChannelData(i));

  while(pos < len) {
    for(i = 0; i < numOfChan; i++) {             // interleave channels
      sample = Math.max(-1, Math.min(1, channels[i][pos])); // clamp
      sample = (0.5 + sample < 0 ? sample * 32768 : sample * 32767)|0; // scale to 16-bit signed int
      view.setInt16(44 + offset, sample, true);          // write 16-bit sample
      offset += 2;
    }
    pos++;
  }

  return new Blob([buffer], {type: "audio/wav"});

  function setUint16(data: any) {
    view.setUint16(pos, data, true);
    pos += 2;
  }

  function setUint32(data: any) {
    view.setUint32(pos, data, true);
    pos += 4;
  }
};

/**
 * Converts raw PCM ArrayBuffer (16-bit, mono) to AudioBuffer.
 * Gemini TTS returns raw PCM at 24kHz by default.
 */
const pcmToAudioBuffer = (buffer: ArrayBuffer, ctx: BaseAudioContext, sampleRate: number = 24000): AudioBuffer => {
  const pcmData = new Int16Array(buffer);
  const audioBuffer = ctx.createBuffer(1, pcmData.length, sampleRate);
  const channelData = audioBuffer.getChannelData(0);

  for (let i = 0; i < pcmData.length; i++) {
    // Convert 16-bit PCM to float [-1.0, 1.0]
    channelData[i] = pcmData[i] / 32768.0;
  }

  return audioBuffer;
};

/**
 * Converts raw PCM data (from Gemini) to a playable WAV Blob.
 * Useful for previews.
 */
export const pcmToWavBlob = (pcmBuffer: ArrayBuffer, sampleRate: number = 24000): Blob => {
  // We need a temporary AudioContext (offline is fine) to decode to AudioBuffer
  // then we use bufferToWave to add headers.
  // Since we don't want to rely on the main thread blocking, using OfflineContext is good.
  // However, pcmToAudioBuffer requires a context.
  
  // Create a minimal offline context just to create the buffer
  const ctx = new OfflineAudioContext(1, 1, sampleRate);
  const audioBuffer = pcmToAudioBuffer(pcmBuffer, ctx, sampleRate);
  
  return bufferToWave(audioBuffer, audioBuffer.length);
};

/**
 * Procedurally generates a backing track buffer based on genre.
 * This ensures the app works without relying on external MP3 URLs that might 404.
 */
export const generateBackingTrackBuffer = async (ctx: BaseAudioContext, track: Track): Promise<AudioBuffer> => {
  const duration = track.duration;
  const sampleRate = ctx.sampleRate;
  const buffer = ctx.createBuffer(2, sampleRate * duration, sampleRate);
  
  // Simple synthesizer logic
  const left = buffer.getChannelData(0);
  const right = buffer.getChannelData(1);
  const tempo = track.bpm;
  const secondsPerBeat = 60 / tempo;

  // Fill with silence initially
  for (let i = 0; i < buffer.length; i++) {
    left[i] = 0;
    right[i] = 0;
  }

  // Helper to add a tone
  const addTone = (freq: number, start: number, len: number, vol: number, type: 'sine' | 'square' | 'sawtooth' | 'noise' = 'sine') => {
      const startSample = Math.floor(start * sampleRate);
      const endSample = Math.floor((start + len) * sampleRate);
      
      for (let i = startSample; i < endSample && i < buffer.length; i++) {
          const t = (i - startSample) / sampleRate;
          let val = 0;
          
          if (type === 'noise') {
             val = (Math.random() * 2 - 1);
          } else {
             val = Math.sin(2 * Math.PI * freq * t);
             if (type === 'square') val = val > 0 ? 1 : -1;
             if (type === 'sawtooth') val = 2 * (t * freq - Math.floor(t * freq + 0.5));
          }

          // Envelope (Fade in/out)
          const fadeLen = 0.05;
          let env = 1;
          if (t < fadeLen) env = t / fadeLen;
          if (t > len - fadeLen) env = (len - t) / fadeLen;

          left[i] += val * vol * env * 0.5; // Left panning
          right[i] += val * vol * env * 0.5; // Right panning
      }
  };

  // Generate pattern based on Genre
  const totalBeats = Math.ceil(duration / secondsPerBeat);
  
  // Frequencies for Christmas melodies
  const N = {
    C4: 261.63,
    D4: 293.66,
    E4: 329.63,
    F4: 349.23,
    G4: 392.00,
    A4: 440.00,
    B4: 493.88,
    C5: 523.25
  };

  // Jingle Bells Melody Pattern (Repeats every 32 beats)
  const jingleBellsMelody = [
    // Measure 1: E E E
    { n: N.E4, b: 0, l: 1 }, { n: N.E4, b: 1, l: 1 }, { n: N.E4, b: 2, l: 2 },
    // Measure 2: E E E
    { n: N.E4, b: 4, l: 1 }, { n: N.E4, b: 5, l: 1 }, { n: N.E4, b: 6, l: 2 },
    // Measure 3: E G C D E
    { n: N.E4, b: 8, l: 1 }, { n: N.G4, b: 9, l: 1 }, { n: N.C4, b: 10, l: 1 }, { n: N.D4, b: 11, l: 1 }, { n: N.E4, b: 12, l: 4 },
    // Measure 4: F F F F F
    { n: N.F4, b: 16, l: 1 }, { n: N.F4, b: 17, l: 1 }, { n: N.F4, b: 18, l: 1 }, { n: N.F4, b: 19, l: 1 }, { n: N.F4, b: 20, l: 1 },
    // Measure 5: E E E E
    { n: N.E4, b: 21, l: 1 }, { n: N.E4, b: 22, l: 1 }, { n: N.E4, b: 23, l: 1 }, { n: N.E4, b: 24, l: 1 },
    // Measure 6: E D D E D G
    { n: N.E4, b: 25, l: 1 }, { n: N.D4, b: 26, l: 1 }, { n: N.D4, b: 27, l: 1 }, { n: N.E4, b: 28, l: 1 }, { n: N.D4, b: 29, l: 2 }, { n: N.G4, b: 31, l: 1 } // Simplified end
  ];

  // Joy to the World (Simplified) for variation (t6)
  const joyMelody = [
    { n: N.C5, b: 0, l: 2 }, 
    { n: N.B4, b: 2, l: 1.5 }, { n: N.A4, b: 3.5, l: 0.5 },
    { n: N.G4, b: 4, l: 3 }, { n: N.F4, b: 7, l: 1 },
    { n: N.E4, b: 8, l: 2 }, { n: N.D4, b: 10, l: 2 },
    { n: N.C4, b: 12, l: 4 },
    // Repeat logic will handle looping, but we can add second phrase if needed
    { n: N.G4, b: 16, l: 1 }, { n: N.A4, b: 17, l: 1 }, { n: N.A4, b: 18, l: 0.5 }, { n: N.B4, b: 18.5, l: 0.5 }, { n: N.B4, b: 19, l: 1 }, { n: N.C5, b: 20, l: 2 }
  ];

  const melodyLoopLength = 32;

  for (let beat = 0; beat < totalBeats; beat++) {
      const time = beat * secondsPerBeat;
      
      if (track.genre === TrackGenre.UPBEAT) {
          // Kick on 1, 2, 3, 4
          addTone(50, time, 0.1, 0.6, 'sine'); // Kick
          // Hat on offbeats
          addTone(0, time + secondsPerBeat/2, 0.05, 0.1, 'noise'); // Hat
          // Chord stab on 1
          if (beat % 4 === 0) {
              addTone(440, time, 0.5, 0.1, 'sawtooth');
              addTone(554, time, 0.5, 0.1, 'sawtooth'); // C#
              addTone(659, time, 0.5, 0.1, 'sawtooth'); // E
          }
      } else if (track.genre === TrackGenre.CORPORATE) {
           // Gentle pulse
           if (beat % 2 === 0) addTone(100, time, 0.1, 0.4, 'sine');
           // Arpeggio
           addTone(300 + (beat % 4) * 50, time, 0.2, 0.05, 'sine');
      } else if (track.genre === TrackGenre.CALM) {
          // Long pads
          if (beat % 8 === 0) {
              addTone(220, time, 4, 0.1, 'sine');
              addTone(330, time, 4, 0.1, 'sine');
          }
          // Chimes
          if (Math.random() > 0.7) {
              addTone(880 + Math.random() * 500, time, 0.3, 0.05, 'sine');
          }
      } else if (track.genre === TrackGenre.DRAMATIC) {
          // Big hits
           if (beat % 4 === 0) {
              addTone(40, time, 0.5, 0.8, 'square'); // Impact
              addTone(40, time, 0.5, 0.8, 'noise'); // Crash
           }
           // Tension strings
           addTone(110, time, secondsPerBeat, 0.1, 'sawtooth');
      } else if (track.genre === TrackGenre.CHRISTMAS) {
          // Sleigh Bells (rhythmic noise) - on every beat and half beat
          addTone(0, time, 0.08, 0.12, 'noise'); 
          addTone(0, time + secondsPerBeat/2, 0.04, 0.08, 'noise');
          
          // Select Melody
          const selectedMelody = track.id === 't6' ? joyMelody : jingleBellsMelody;

          // Play Melody
          const beatInLoop = beat % melodyLoopLength;
          // Find notes starting on this beat
          const notes = selectedMelody.filter(n => n.b === beatInLoop);
          
          notes.forEach(note => {
             // Play note
             addTone(note.n, time, note.l * secondsPerBeat * 0.9, 0.15, 'sine');
             // Add a bit of harmonics for "bell" sound
             addTone(note.n * 2, time, note.l * secondsPerBeat * 0.5, 0.05, 'sine');
          });

          // Simple Bass on main beats
          if (beat % 4 === 0) {
              addTone(N.C4 / 2, time, 0.3, 0.15, 'sawtooth'); // C3
          } else if (beat % 4 === 2) {
              addTone(N.G4 / 2, time, 0.3, 0.15, 'sawtooth'); // G3
          }
      }
  }

  return buffer;
};

/**
 * Gets a playable URL for the track preview.
 * Returns the file URL if available, or generates a short procedural clip.
 */
export const getTrackPreviewUrl = async (track: Track): Promise<string> => {
  if (track.fileUrl) {
    try {
      // Check if file exists before trying to use it.
      // If the file is missing (404), fetch will throw or return !ok.
      const response = await fetch(track.fileUrl);
      if (response.ok) {
        return track.fileUrl;
      }
    } catch (e) {
      console.warn(`Preview file ${track.fileUrl} missing or invalid, generating procedural preview instead.`);
    }
  }
  
  // Generate procedural preview (shortened)
  const sampleRate = 44100;
  const offlineCtx = new OfflineAudioContext(2, sampleRate * 10, sampleRate); // 10s preview
  
  // Create a copy of the track with 10s duration for faster generation
  const previewTrack = { ...track, duration: 10 };
  
  const buffer = await generateBackingTrackBuffer(offlineCtx, previewTrack);
  const wavBlob = bufferToWave(buffer, buffer.length);
  return URL.createObjectURL(wavBlob);
};

/**
 * Mixes the voiceover with the backing track.
 * Implements ducking (-40%) and fade out.
 */
export const mixAudioTracks = async (
  track: Track,
  voiceArrayBuffer: ArrayBuffer
): Promise<string> => {
  // 1. Setup Context
  // Gemini TTS is 24kHz, backing tracks are synthetic. Standardizing on 44.1kHz for output.
  const offlineCtx = new OfflineAudioContext(2, 44100 * track.duration, 44100);

  // 2. Load Voice
  // Native decodeAudioData fails on raw PCM from Gemini. We must manually decode it.
  // Gemini returns 24000Hz mono PCM.
  const voiceBuffer = pcmToAudioBuffer(voiceArrayBuffer, offlineCtx, 24000);

  // 3. Generate OR Fetch Track Buffer
  let trackBuffer: AudioBuffer;

  if (track.fileUrl) {
    try {
      // Attempt to load the specific file requested
      const response = await fetch(track.fileUrl);
      if (!response.ok) throw new Error("File not found");
      const arrayBuffer = await response.arrayBuffer();
      // decodeAudioData requires the buffer to be in a format browser understands (MP3/WAV)
      trackBuffer = await offlineCtx.decodeAudioData(arrayBuffer);
    } catch (e) {
      console.warn(`Could not load external file ${track.fileUrl}, falling back to generator.`, e);
      trackBuffer = await generateBackingTrackBuffer(offlineCtx, track);
    }
  } else {
    trackBuffer = await generateBackingTrackBuffer(offlineCtx, track);
  }

  // 4. Create Sources
  const trackSource = offlineCtx.createBufferSource();
  trackSource.buffer = trackBuffer;
  
  const voiceSource = offlineCtx.createBufferSource();
  voiceSource.buffer = voiceBuffer;

  // 5. Create Gain Nodes
  const trackGain = offlineCtx.createGain();
  const voiceGain = offlineCtx.createGain();
  
  // 6. Connect Graph
  trackSource.connect(trackGain);
  trackGain.connect(offlineCtx.destination);

  voiceSource.connect(voiceGain);
  voiceGain.connect(offlineCtx.destination);

  // 7. Scheduling & Automation
  const now = 0;
  const voiceStart = track.cuePoint;
  const voiceDuration = voiceBuffer.duration;
  const voiceEnd = voiceStart + voiceDuration;

  // Track starts at 0
  trackSource.start(now);
  
  // Voice starts at cue point
  voiceSource.start(voiceStart);

  // -- DUCKING AUTOMATION --
  // Base volume
  trackGain.gain.setValueAtTime(0.8, now); // Initial volume slightly lower to prevent clipping

  // Ramp down to 40% (0.4 relative to max, user asked for -40% so ~0.6 of original)
  // Let's say original is 0.8. -40% is approx 0.48.
  const duckVolume = 0.3; 
  const rampTime = 0.5; // 500ms fade

  // Duck down just before voice starts
  trackGain.gain.setValueAtTime(0.8, Math.max(0, voiceStart - rampTime));
  trackGain.gain.linearRampToValueAtTime(duckVolume, voiceStart);
  
  // Hold volume down until voice ends
  trackGain.gain.setValueAtTime(duckVolume, voiceEnd);
  
  // Ramp back up
  trackGain.gain.linearRampToValueAtTime(0.8, voiceEnd + rampTime);

  // -- FADE OUT --
  const fadeOutStart = track.duration - 2.0;
  trackGain.gain.setValueAtTime(0.8, fadeOutStart);
  trackGain.gain.linearRampToValueAtTime(0, track.duration);

  // 8. Render
  const renderedBuffer = await offlineCtx.startRendering();

  // 9. Convert to Blob/URL
  const wavBlob = bufferToWave(renderedBuffer, renderedBuffer.length);
  return URL.createObjectURL(wavBlob);
};