// Este serviço agora atua como uma ponte (Client) para as APIs Serverless da pasta /api
// Isso protege sua API_KEY, que agora fica segura no servidor (Vercel) e não no navegador.

/**
 * Envia o texto para o backend para ser polido pela IA.
 */
export const polishScript = async (rawText: string, trackName: string): Promise<string> => {
  try {
    const response = await fetch('/api/generate-script', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({ rawText, trackName }),
    });

    if (!response.ok) {
      const errData = await response.json();
      throw new Error(errData.error || 'Falha ao conectar com o servidor');
    }

    const data = await response.json();
    return data.text;
  } catch (error) {
    console.error("Erro ao polir script:", error);
    // Fallback: se a API falhar (ex: rodando local sem servidor), retorna o texto original
    return rawText;
  }
};

/**
 * Solicita ao backend a geração de áudio (TTS).
 */
export const generateSpeech = async (text: string, voiceName: string = 'Kore'): Promise<ArrayBuffer> => {
  const response = await fetch('/api/generate-speech', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({ text, voiceName }),
  });

  if (!response.ok) {
    const errData = await response.json().catch(() => ({}));
    throw new Error(errData.error || 'Falha na geração de voz');
  }

  const data = await response.json();
  const base64Audio = data.audioData;

  // Decode Base64 to ArrayBuffer
  const binaryString = atob(base64Audio);
  const len = binaryString.length;
  const bytes = new Uint8Array(len);
  for (let i = 0; i < len; i++) {
    bytes[i] = binaryString.charCodeAt(i);
  }
  return bytes.buffer;
};

/**
 * Envia áudio do microfone para o backend transcrever.
 */
export const transcribeAudio = async (audioBlob: Blob): Promise<string> => {
  // Convert Blob to Base64 first
  const reader = new FileReader();
  const base64Promise = new Promise<string>((resolve, reject) => {
    reader.onloadend = () => {
      const result = reader.result as string;
      // Remove o prefixo "data:audio/webm;base64," se existir
      const base64String = result.split(',')[1];
      resolve(base64String);
    };
    reader.onerror = reject;
    reader.readAsDataURL(audioBlob);
  });
  
  const base64Data = await base64Promise;

  const response = await fetch('/api/transcribe', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({ 
      audioData: base64Data,
      mimeType: audioBlob.type || 'audio/webm'
    }),
  });

  if (!response.ok) {
    throw new Error('Falha na transcrição');
  }

  const data = await response.json();
  return data.text;
};