export enum AppStep {
  SELECT_TRACK = 0,
  INPUT_SCRIPT = 1,
  PROCESSING = 2,
  RESULT = 3,
}

export enum TrackGenre {
  UPBEAT = 'UPBEAT',
  CALM = 'CALM',
  CORPORATE = 'CORPORATE',
  DRAMATIC = 'DRAMATIC',
  CHRISTMAS = 'CHRISTMAS',
  POLITICAL = 'POLITICAL', // Novo gênero
}

export enum TrackCategory {
  STORE = 'STORE',
  CANDIDATE = 'CANDIDATE',
}

export interface Track {
  id: string;
  name: string;
  genre: TrackGenre;
  category: TrackCategory; // Novo campo
  description: string;
  cuePoint: number; // Seconds where voice should start
  duration: number; // Total duration in seconds
  bpm: number;
  color: string;
  suggestedScript?: string; // Optional default text for this track
  fileUrl?: string; // Optional: URL path to a specific audio file (mp3/wav)
}

export interface Voice {
  id: string; // 'Puck' | 'Charon' | 'Kore' | 'Fenrir' | 'Zephyr'
  name: string;
  gender: 'Masculino' | 'Feminino';
  style: string;
  description: string;
}

export interface JingleData {
  trackId: string;
  originalScript: string;
  polishedScript: string;
  voiceUrl: string;
  finalMixUrl: string;
}