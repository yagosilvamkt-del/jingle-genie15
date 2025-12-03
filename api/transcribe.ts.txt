import { GoogleGenAI } from "@google/genai";

export const config = {
  runtime: 'edge',
};

export default async function handler(req: Request) {
  if (req.method !== 'POST') {
    return new Response('Method Not Allowed', { status: 405 });
  }

  try {
    const { audioData, mimeType } = await req.json();
    const apiKey = process.env.API_KEY;

    if (!apiKey) {
      return new Response(JSON.stringify({ error: 'Chave de API não configurada no servidor' }), { status: 500 });
    }

    const ai = new GoogleGenAI({ apiKey });

    const response = await ai.models.generateContent({
      model: 'gemini-2.5-flash',
      contents: {
        parts: [
          {
            inlineData: {
              mimeType: mimeType || 'audio/webm',
              data: audioData
            }
          },
          {
            text: "Transcreva este áudio exatamente como foi falado em Português."
          }
        ]
      }
    });

    const transcription = response.text?.trim() || "";

    return new Response(JSON.stringify({ text: transcription }), {
      status: 200,
      headers: { 'Content-Type': 'application/json' }
    });

  } catch (error: any) {
    console.error('Erro na API transcribe:', error);
    return new Response(JSON.stringify({ error: error.message }), { status: 500 });
  }
}