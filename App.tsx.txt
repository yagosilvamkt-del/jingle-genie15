import React, { useState, useRef, useEffect } from 'react';
import { Play, Pause, Music, Mic, Wand2, Download, ChevronRight, RotateCcw, Check, Sparkles, AlertCircle, Volume2, StopCircle, Edit3, Lock, CreditCard, X, QrCode, Store, Megaphone, ExternalLink, Video, CheckCircle2, Instagram, MessageCircle } from 'lucide-react';
import { AppStep, Track, JingleData, TrackGenre, Voice, TrackCategory } from './types';
import { TRACK_LIBRARY, VOICE_LIBRARY, SCRIPT_TEMPLATES } from './constants';
import Button from './components/Button';
import { polishScript, generateSpeech, transcribeAudio } from './services/geminiService';
import { mixAudioTracks, pcmToWavBlob, getTrackPreviewUrl } from './services/audioService';

const GENRE_LABELS: Record<TrackGenre, string> = {
  [TrackGenre.UPBEAT]: 'AGITADO',
  [TrackGenre.CALM]: 'CALMO',
  [TrackGenre.CORPORATE]: 'CORPORATIVO',
  [TrackGenre.DRAMATIC]: 'DRAMÁTICO',
  [TrackGenre.CHRISTMAS]: 'NATALINO',
  [TrackGenre.POLITICAL]: 'POLÍTICO',
};

// Preços Atualizados
const PRICE_JINGLE = 49.00;
const PRICE_VIDEO_UPSELL = 29.99;

// CONFIGURAÇÃO: COLOQUE SEU NÚMERO AQUI (com DDI 55 e DDD)
// Exemplo: '5511999999999'
const WHATSAPP_NUMBER = '5511999999999';

// Helper for formatting time seconds -> mm:ss
const formatTime = (time: number) => {
  if (isNaN(time)) return "00:00";
  const minutes = Math.floor(time / 60);
  const seconds = Math.floor(time % 60);
  return `${minutes.toString().padStart(2, '0')}:${seconds.toString().padStart(2, '0')}`;
};

export default function App() {
  const [step, setStep] = useState<AppStep>(AppStep.SELECT_TRACK);
  const [selectedCategory, setSelectedCategory] = useState<TrackCategory>(TrackCategory.STORE);
  const [selectedTrack, setSelectedTrack] = useState<Track | null>(null);
  const [selectedVoice, setSelectedVoice] = useState<Voice>(VOICE_LIBRARY[2]); // Default 'Kore'
  const [rawScript, setRawScript] = useState('');
  const [useAiPolish, setUseAiPolish] = useState(true); // New state for AI toggle
  const [isRecording, setIsRecording] = useState(false);
  const [isProcessing, setIsProcessing] = useState(false);
  const [processingStatus, setProcessingStatus] = useState('');
  const [jingleData, setJingleData] = useState<JingleData | null>(null);
  const [error, setError] = useState<string | null>(null);
  const [isPlaying, setIsPlaying] = useState(false);
  
  // Audio Player State
  const [currentTime, setCurrentTime] = useState(0);
  const [duration, setDuration] = useState(0);
  
  // Payment & Upsell State
  const [showPaymentModal, setShowPaymentModal] = useState(false);
  const [isPaid, setIsPaid] = useState(false);
  const [upsellSelected, setUpsellSelected] = useState(false);

  // Track Preview State (Step 1)
  const [playingTrackId, setPlayingTrackId] = useState<string | null>(null);

  // Voice Preview State (Step 2)
  const [previewingVoiceId, setPreviewingVoiceId] = useState<string | null>(null);

  // Audio Refs
  const audioRef = useRef<HTMLAudioElement | null>(null);
  const mediaRecorderRef = useRef<MediaRecorder | null>(null);
  const chunksRef = useRef<Blob[]>([]);
  const previewAudioRef = useRef<HTMLAudioElement | null>(null);

  // Check URL parameters for payment success (Integration Logic)
  useEffect(() => {
    const params = new URLSearchParams(window.location.search);
    const status = params.get('status');
    const trackId = params.get('track_id'); // Optional: restore state

    if (status === 'paid' || status === 'approved' || status === 'success') {
       // In a real app, you might verify this against your backend session
       setIsPaid(true);
       
       // Clean URL
       window.history.replaceState({}, document.title, window.location.pathname);
       
       // Optional: Restore UI state if coming back from redirect
       // (Requires saving state to localStorage before redirecting)
    }
  }, []);

  // Cleanup audio URL on unmount
  useEffect(() => {
    return () => {
      if (jingleData?.finalMixUrl) {
        URL.revokeObjectURL(jingleData.finalMixUrl);
      }
    };
  }, [jingleData]);

  const handleTrackSelect = (track: Track) => {
    // Stop any previews
    if (previewAudioRef.current) {
        previewAudioRef.current.pause();
        setPlayingTrackId(null);
    }
    
    setSelectedTrack(track);
    // Pre-fill with suggested script if available, otherwise clear
    setRawScript(track.suggestedScript || '');
    setStep(AppStep.INPUT_SCRIPT);
  };

  const handleToggleTrackPreview = async (e: React.MouseEvent, track: Track) => {
    e.stopPropagation();
    
    // Stop if playing this track
    if (playingTrackId === track.id) {
      previewAudioRef.current?.pause();
      setPlayingTrackId(null);
      return;
    }

    // Stop currently playing track if any
    if (previewAudioRef.current) {
      previewAudioRef.current.pause();
    }

    setPlayingTrackId(track.id); // Set loading/playing state immediately to verify UI interaction

    try {
      const url = await getTrackPreviewUrl(track);
      
      const audio = new Audio(url);
      
      audio.onended = () => {
         setPlayingTrackId(null);
         if (url.startsWith('blob:')) {
            URL.revokeObjectURL(url);
         }
      };
      
      audio.onerror = () => {
         console.error("Error loading track preview");
         setError(`Não foi possível carregar a prévia da trilha: ${track.name}. Verifique o arquivo.`);
         setPlayingTrackId(null);
      };

      await audio.play();
      previewAudioRef.current = audio;
      
    } catch (err) {
      console.error("Preview failed", err);
      setError("Erro ao gerar ou carregar prévia.");
      setPlayingTrackId(null);
    }
  };

  const handlePreviewVoice = async (voice: Voice) => {
     // Stop any current playback
     if (previewAudioRef.current) {
        previewAudioRef.current.pause();
        previewAudioRef.current = null;
        setPlayingTrackId(null); // Ensure track preview state is cleared too if using same ref
     }
     
     if (previewingVoiceId === voice.id) {
        setPreviewingVoiceId(null);
        return;
     }

     setPreviewingVoiceId(voice.id);
     
     try {
       const text = `Esta é a voz de ${voice.name}. O que você acha?`;
       const buffer = await generateSpeech(text, voice.id);
       const blob = pcmToWavBlob(buffer);
       const url = URL.createObjectURL(blob);
       
       const audio = new Audio(url);
       previewAudioRef.current = audio;
       
       audio.onended = () => {
         setPreviewingVoiceId(null);
         URL.revokeObjectURL(url);
       };
       
       audio.play();
     } catch (err) {
       console.error(err);
       setError("Não foi possível carregar a prévia da voz.");
       setPreviewingVoiceId(null);
     }
  };

  const handleApplyTemplate = (templateId: string, text: string) => {
    if (templateId === 'random') {
       // Pick a random template excluding the 'random' one itself
       const validTemplates = SCRIPT_TEMPLATES.filter(t => t.id !== 'random');
       const randomTemplate = validTemplates[Math.floor(Math.random() * validTemplates.length)];
       setRawScript(randomTemplate.text);
    } else {
       setRawScript(text);
    }
  };

  const handleStartRecording = async () => {
    try {
      const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
      const mediaRecorder = new MediaRecorder(stream);
      mediaRecorderRef.current = mediaRecorder;
      chunksRef.current = [];

      mediaRecorder.ondataavailable = (e) => {
        if (e.data.size > 0) chunksRef.current.push(e.data);
      };

      mediaRecorder.onstop = async () => {
        const audioBlob = new Blob(chunksRef.current, { type: 'audio/webm' });
        setIsProcessing(true);
        setProcessingStatus('Transcrevendo sua voz...');
        try {
            const transcription = await transcribeAudio(audioBlob);
            setRawScript(transcription);
        } catch (e) {
            setError("Falha ao transcrever áudio. Tente novamente.");
            console.error(e);
        } finally {
            setIsProcessing(false);
        }
        
        // Stop all tracks
        stream.getTracks().forEach(track => track.stop());
      };

      mediaRecorder.start();
      setIsRecording(true);
    } catch (err) {
      setError("Acesso ao microfone negado ou indisponível.");
    }
  };

  const handleStopRecording = () => {
    if (mediaRecorderRef.current && isRecording) {
      mediaRecorderRef.current.stop();
      setIsRecording(false);
    }
  };

  const processJingle = async () => {
    if (!selectedTrack || !rawScript.trim()) return;

    setIsProcessing(true);
    setStep(AppStep.PROCESSING);
    setError(null);
    // Note: Do not reset isPaid here if you want to allow re-generations under the same license session
    // For now we reset to simulate a new product flow
    setIsPaid(false);
    setUpsellSelected(false); // Reset upsell

    // Stop any previews
    if (previewAudioRef.current) {
        previewAudioRef.current.pause();
    }

    try {
      let polished = rawScript;

      // 1. Polish Script (Only if enabled)
      if (useAiPolish) {
        setProcessingStatus('IA está polindo seu texto...');
        polished = await polishScript(rawScript, selectedTrack.name);
      } else {
        setProcessingStatus('Preparando texto original...');
      }
      
      // 2. Generate Speech
      setProcessingStatus('Gerando locução profissional...');
      const voiceBuffer = await generateSpeech(polished, selectedVoice.id);

      // 3. Mix Audio
      setProcessingStatus('Mixando faixas de áudio...');
      // Create a temporary object URL for the voice just for the record
      const voiceBlob = new Blob([voiceBuffer], { type: 'audio/mp3' }); 
      const voiceUrl = URL.createObjectURL(voiceBlob);

      const finalMixUrl = await mixAudioTracks(selectedTrack, voiceBuffer);

      setJingleData({
        trackId: selectedTrack.id,
        originalScript: rawScript,
        polishedScript: polished,
        voiceUrl,
        finalMixUrl
      });

      setStep(AppStep.RESULT);

    } catch (err: any) {
      console.error(err);
      setError(err.message || "Ocorreu um erro inesperado.");
      setStep(AppStep.INPUT_SCRIPT);
    } finally {
      setIsProcessing(false);
    }
  };

  // Player Controls
  const togglePlay = () => {
    if (!audioRef.current || !jingleData) return;
    
    if (isPlaying) {
      audioRef.current.pause();
    } else {
      audioRef.current.play();
    }
    setIsPlaying(!isPlaying);
  };

  const handleTimeUpdate = () => {
    if (audioRef.current) {
      setCurrentTime(audioRef.current.currentTime);
    }
  };

  const handleLoadedMetadata = () => {
    if (audioRef.current) {
      setDuration(audioRef.current.duration);
    }
  };

  const handleSeek = (e: React.ChangeEvent<HTMLInputElement>) => {
    if (audioRef.current) {
      const time = Number(e.target.value);
      audioRef.current.currentTime = time;
      setCurrentTime(time);
    }
  };

  const resetApp = () => {
    setStep(AppStep.SELECT_TRACK);
    setSelectedTrack(null);
    setRawScript('');
    setJingleData(null);
    setError(null);
    setIsPlaying(false);
    setIsPaid(false);
    setUpsellSelected(false);
    setCurrentTime(0);
    setDuration(0);
  };

  // Payment Handlers
  const getTotalPrice = () => {
     return upsellSelected ? PRICE_JINGLE + PRICE_VIDEO_UPSELL : PRICE_JINGLE;
  };

  const handleBuyClick = () => {
    setShowPaymentModal(true);
  };

  const handleConfirmPayment = () => {
    // Mock Payment Confirmation
    setTimeout(() => {
      setIsPaid(true);
      setShowPaymentModal(false);
    }, 1500);
  };
  
  const handleCaktoRedirect = () => {
      // Exemplo de integração: Redirecionar para seu Checkout Cakto
      const MOCK_CHECKOUT_URL = window.location.href.split('?')[0] + "?status=paid";
      alert(`Em produção, redirecionaria para Cakto. Valor: R$ ${getTotalPrice().toFixed(2)}`);
      window.location.href = MOCK_CHECKOUT_URL;
  };

  // WhatsApp Link Builder
  const getWhatsAppLink = () => {
    const text = `Olá! Acabei de comprar o pacote Jingle + Vídeo e gostaria de enviar minhas fotos para a edição.\n\nNúmero do Pedido: #${Math.floor(Math.random() * 10000)}`;
    return `https://wa.me/${WHATSAPP_NUMBER}?text=${encodeURIComponent(text)}`;
  };

  // Filter tracks by category
  const filteredTracks = TRACK_LIBRARY.filter(t => t.category === selectedCategory);

  return (
    <div className="min-h-screen bg-gray-900 text-gray-100 flex flex-col items-center p-4 md:p-8">
      
      {/* Header */}
      <header className="w-full max-w-4xl flex items-center justify-between mb-12">
        <div className="flex items-center gap-3">
          <div className="bg-gradient-to-br from-indigo-500 to-purple-600 p-2 rounded-lg">
            <Music className="w-6 h-6 text-white" />
          </div>
          <div>
            <h1 className="text-2xl font-bold bg-clip-text text-transparent bg-gradient-to-r from-indigo-400 to-purple-400">
              JingleGenie
            </h1>
            <p className="text-xs text-gray-400">Criador de Comerciais com IA</p>
          </div>
        </div>
        
        {/* Progress Steps */}
        <div className="hidden md:flex items-center gap-2">
          {[AppStep.SELECT_TRACK, AppStep.INPUT_SCRIPT, AppStep.RESULT].map((s, idx) => (
            <div key={idx} className="flex items-center">
              <div className={`w-8 h-8 rounded-full flex items-center justify-center text-sm font-bold ${
                step >= s ? 'bg-indigo-600 text-white' : 'bg-gray-800 text-gray-500'
              }`}>
                {idx + 1}
              </div>
              {idx < 2 && <div className={`w-8 h-1 ${step > s ? 'bg-indigo-600' : 'bg-gray-800'}`} />}
            </div>
          ))}
        </div>
      </header>

      <main className="w-full max-w-4xl bg-gray-800/50 backdrop-blur-sm border border-gray-700 rounded-3xl p-6 md:p-10 shadow-2xl min-h-[500px] flex flex-col relative overflow-hidden">
        
        {/* Error Banner */}
        {error && (
          <div className="mb-6 p-4 bg-red-900/30 border border-red-500/50 rounded-xl flex items-center gap-3 text-red-200 animate-fade-in z-20 relative">
            <AlertCircle className="w-5 h-5 flex-shrink-0" />
            <p className="text-sm">{error}</p>
            <button onClick={() => setError(null)} className="ml-auto hover:text-white">✕</button>
          </div>
        )}

        {/* STEP 1: SELECT CATEGORY & TRACK */}
        {step === AppStep.SELECT_TRACK && (
          <div className="flex flex-col h-full animate-fade-in">
            <h2 className="text-3xl font-bold mb-6 text-center">Escolha o modelo de sua locução</h2>
            
            {/* Category Toggle */}
            <div className="flex justify-center mb-8">
               <div className="bg-gray-700 p-1 rounded-xl inline-flex">
                  <button 
                     onClick={() => setSelectedCategory(TrackCategory.STORE)}
                     className={`flex items-center gap-2 px-6 py-2 rounded-lg font-medium transition-all duration-200 ${
                        selectedCategory === TrackCategory.STORE 
                        ? 'bg-indigo-600 text-white shadow-lg' 
                        : 'text-gray-400 hover:text-white'
                     }`}
                  >
                     <Store className="w-4 h-4" /> Para Lojas
                  </button>
                  <button 
                     onClick={() => setSelectedCategory(TrackCategory.CANDIDATE)}
                     className={`flex items-center gap-2 px-6 py-2 rounded-lg font-medium transition-all duration-200 ${
                        selectedCategory === TrackCategory.CANDIDATE 
                        ? 'bg-indigo-600 text-white shadow-lg' 
                        : 'text-gray-400 hover:text-white'
                     }`}
                  >
                     <Megaphone className="w-4 h-4" /> Candidatos
                  </button>
               </div>
            </div>

            <div className="mb-4">
               <h3 className="text-lg font-semibold text-gray-300 mb-2">Escolha a trilha de fundo:</h3>
               <p className="text-gray-400 text-sm">Selecione o estilo musical que acompanhará sua mensagem.</p>
            </div>
            
            <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
              {filteredTracks.map((track) => (
                <div
                  key={track.id}
                  onClick={() => handleTrackSelect(track)}
                  className="group relative p-6 bg-gray-700/50 hover:bg-gray-700 border border-gray-600 hover:border-indigo-500 rounded-2xl text-left transition-all duration-300 hover:scale-[1.02] hover:shadow-xl cursor-pointer"
                >
                  <div className={`absolute top-0 right-0 w-24 h-24 ${track.color} opacity-10 rounded-full blur-2xl -mr-10 -mt-10 transition-opacity group-hover:opacity-20`} />
                  
                  <div className="flex items-center justify-between mb-4">
                    <span className={`text-xs font-bold px-2 py-1 rounded-full bg-gray-800 text-gray-300 uppercase tracking-wider`}>
                      {GENRE_LABELS[track.genre]}
                    </span>
                    
                    {/* Preview Button inside card */}
                    <button
                        onClick={(e) => handleToggleTrackPreview(e, track)}
                        className={`w-10 h-10 rounded-full flex items-center justify-center transition-all duration-200 z-10 ${
                            playingTrackId === track.id
                            ? 'bg-indigo-500 text-white shadow-lg scale-110'
                            : 'bg-gray-800 text-gray-400 hover:bg-indigo-600 hover:text-white'
                        }`}
                        title={playingTrackId === track.id ? "Parar prévia" : "Ouvir prévia"}
                    >
                        {playingTrackId === track.id ? (
                            <StopCircle className="w-5 h-5 fill-current" />
                        ) : (
                            <Play className="w-5 h-5 fill-current ml-0.5" />
                        )}
                    </button>
                  </div>
                  
                  <h3 className="text-xl font-bold mb-1 group-hover:text-indigo-300 transition-colors">{track.name}</h3>
                  <p className="text-sm text-gray-400">{track.description}</p>
                  
                  <div className="mt-4 flex items-center gap-2 text-xs text-gray-500">
                     <span className="flex items-center gap-1">
                        <div className="w-1.5 h-1.5 bg-green-500 rounded-full"></div>
                        Voz em {track.cuePoint}s
                     </span>
                     <span>•</span>
                     <span>{track.duration}s total</span>
                  </div>
                </div>
              ))}
            </div>
          </div>
        )}

        {/* STEP 2: INPUT SCRIPT & VOICE */}
        {step === AppStep.INPUT_SCRIPT && selectedTrack && (
          <div className="flex flex-col h-full animate-fade-in space-y-8">
            <div className="flex justify-between items-center">
              <button onClick={() => setStep(AppStep.SELECT_TRACK)} className="text-gray-400 hover:text-white flex items-center gap-1 text-sm w-fit transition-colors">
                ← Voltar para trilhas
              </button>
              <span className="text-indigo-400 font-medium text-sm">Trilha: {selectedTrack.name}</span>
            </div>

            {/* Script Input Section */}
            <div>
               <h2 className="text-2xl font-bold mb-2">O que você está anunciando?</h2>
               <p className="text-gray-400 text-sm mb-4">Escreva seu anúncio ou escolha um modelo pronto:</p>
               
               {/* Quick Templates Buttons */}
               <div className="flex gap-2 overflow-x-auto pb-4 mb-2 scrollbar-thin scrollbar-thumb-gray-600">
                  {SCRIPT_TEMPLATES.map((tpl) => (
                    <button
                      key={tpl.id}
                      onClick={() => handleApplyTemplate(tpl.id, tpl.text)}
                      className="flex items-center gap-2 px-4 py-2 bg-gray-700/50 hover:bg-indigo-600 border border-gray-600 hover:border-indigo-500 rounded-full text-xs font-medium text-gray-300 hover:text-white transition-all whitespace-nowrap active:scale-95"
                    >
                      {tpl.label}
                    </button>
                  ))}
               </div>

               <div className="relative">
                  <textarea
                    value={rawScript}
                    onChange={(e) => setRawScript(e.target.value)}
                    placeholder="Ex: Estamos fazendo uma grande liquidação nesta sexta-feira na Loja do João. 50% de desconto em todas as ferramentas. Não perca."
                    className="w-full h-32 bg-gray-900/50 border border-gray-600 rounded-2xl p-4 text-lg focus:ring-2 focus:ring-indigo-500 focus:border-transparent outline-none resize-none transition-all placeholder:text-gray-600"
                  />
                  <div className="absolute bottom-3 right-3">
                     <button
                        onClick={isRecording ? handleStopRecording : handleStartRecording}
                        disabled={isProcessing}
                        className={`p-2 rounded-full transition-all ${
                          isRecording 
                            ? 'bg-red-500 animate-pulse text-white shadow-red-500/50 shadow-lg' 
                            : 'bg-gray-700 hover:bg-gray-600 text-gray-300'
                        }`}
                        title="Gravar áudio"
                     >
                        <Mic className="w-4 h-4" />
                     </button>
                  </div>
               </div>

               {/* AI Polish Toggle */}
               <div className="mt-3 flex items-center gap-3">
                 <button 
                   onClick={() => setUseAiPolish(!useAiPolish)}
                   className={`relative inline-flex h-6 w-11 items-center rounded-full transition-colors focus:outline-none focus:ring-2 focus:ring-indigo-500 focus:ring-offset-2 focus:ring-offset-gray-900 ${useAiPolish ? 'bg-indigo-600' : 'bg-gray-700'}`}
                 >
                   <span
                     className={`${useAiPolish ? 'translate-x-6' : 'translate-x-1'} inline-block h-4 w-4 transform rounded-full bg-white transition-transform`}
                   />
                 </button>
                 <div className="flex items-center gap-2 cursor-pointer" onClick={() => setUseAiPolish(!useAiPolish)}>
                   <Edit3 className={`w-4 h-4 ${useAiPolish ? 'text-indigo-400' : 'text-gray-500'}`} />
                   <span className={`text-sm select-none ${useAiPolish ? 'text-indigo-200 font-medium' : 'text-gray-500'}`}>
                     Otimizar texto com IA (Reescrever para estilo comercial)
                   </span>
                 </div>
               </div>
            </div>

            {/* Voice Selection Section */}
            <div>
               <h2 className="text-2xl font-bold mb-2">Escolha a Voz</h2>
               <p className="text-gray-400 text-sm mb-4">Selecione o narrador ideal para o seu comercial.</p>
               
               <div className="grid grid-cols-1 sm:grid-cols-2 gap-3">
                  {VOICE_LIBRARY.map((voice) => (
                     <div 
                        key={voice.id}
                        className={`p-3 rounded-xl border cursor-pointer transition-all flex items-center justify-between ${
                           selectedVoice.id === voice.id 
                           ? 'bg-indigo-600/20 border-indigo-500 shadow-lg shadow-indigo-900/20' 
                           : 'bg-gray-800 border-gray-700 hover:border-gray-500'
                        }`}
                        onClick={() => setSelectedVoice(voice)}
                     >
                        <div className="flex items-center gap-3">
                           <div className={`w-8 h-8 rounded-full flex items-center justify-center text-xs font-bold ${
                              selectedVoice.id === voice.id ? 'bg-indigo-500 text-white' : 'bg-gray-700 text-gray-400'
                           }`}>
                              {voice.name[0]}
                           </div>
                           <div>
                              <p className={`font-semibold text-sm ${selectedVoice.id === voice.id ? 'text-white' : 'text-gray-300'}`}>
                                 {voice.name}
                              </p>
                              <p className="text-xs text-gray-500">{voice.style} • {voice.gender}</p>
                           </div>
                        </div>
                        
                        <button 
                           onClick={(e) => { e.stopPropagation(); handlePreviewVoice(voice); }}
                           className={`p-2 rounded-full transition-colors ${
                              previewingVoiceId === voice.id 
                              ? 'bg-indigo-500 text-white' 
                              : 'bg-gray-700 hover:bg-gray-600 text-gray-400'
                           }`}
                           title="Ouvir prévia"
                        >
                           {previewingVoiceId === voice.id ? <StopCircle className="w-4 h-4" /> : <Volume2 className="w-4 h-4" />}
                        </button>
                     </div>
                  ))}
               </div>
            </div>

            <div className="flex items-center justify-end pt-4">
               <div className="flex flex-col items-end gap-2 w-full md:w-auto">
                 <p className="text-xs text-gray-500 text-right">
                    Voz selecionada: <strong className="text-gray-300">{selectedVoice.name}</strong>
                 </p>
                 <Button 
                    onClick={processJingle} 
                    disabled={!rawScript.trim() || isProcessing}
                    isLoading={isProcessing}
                    className="w-full md:w-auto"
                 >
                    <Sparkles className="w-5 h-5" />
                    Criar Jingle Mágico
                 </Button>
               </div>
            </div>
          </div>
        )}

        {/* STEP 3: PROCESSING STATE */}
        {step === AppStep.PROCESSING && (
           <div className="flex flex-col items-center justify-center h-full min-h-[400px] animate-fade-in text-center">
              <div className="relative w-24 h-24 mb-8">
                 <div className="absolute inset-0 border-4 border-indigo-500/30 rounded-full animate-ping"></div>
                 <div className="absolute inset-0 border-4 border-indigo-500 rounded-full border-t-transparent animate-spin"></div>
                 <Wand2 className="absolute inset-0 m-auto text-indigo-400 w-8 h-8 animate-bounce" />
              </div>
              <h3 className="text-2xl font-bold text-white mb-2">Gerando Jingle</h3>
              <p className="text-indigo-300 animate-pulse">{processingStatus}</p>
           </div>
        )}

        {/* STEP 4: RESULT */}
        {step === AppStep.RESULT && jingleData && (
          <div className="flex flex-col h-full animate-fade-in">
            <div className="flex items-center justify-between mb-8">
              <h2 className="text-3xl font-bold bg-clip-text text-transparent bg-gradient-to-r from-green-400 to-emerald-500">
                 Está Pronto!
              </h2>
              <button onClick={resetApp} className="text-gray-400 hover:text-white flex items-center gap-2 text-sm transition-colors">
                <RotateCcw className="w-4 h-4" /> Criar Novo
              </button>
            </div>

            {/* Script Text */}
            <div className="bg-gray-800/50 p-4 rounded-xl border border-gray-700 mb-6">
                 <h4 className="text-xs font-bold text-indigo-400 uppercase tracking-widest mb-2">Roteiro Final</h4>
                 <p className="text-lg leading-relaxed text-gray-200">
                     "{jingleData.polishedScript}"
                 </p>
            </div>

            {/* Audio Player Container */}
            <div className="bg-gray-900 rounded-2xl p-6 border border-gray-700 mb-6 shadow-xl">
               <div className="flex items-center gap-4 mb-4">
                  {/* Play Button */}
                  <button 
                     onClick={togglePlay}
                     className="w-14 h-14 rounded-full bg-gradient-to-br from-indigo-500 to-purple-600 hover:from-indigo-400 hover:to-purple-500 shadow-lg shadow-indigo-500/30 flex items-center justify-center transition-transform hover:scale-105 active:scale-95 flex-shrink-0"
                  >
                     {isPlaying ? (
                        <Pause className="w-6 h-6 text-white fill-current" />
                     ) : (
                        <Play className="w-6 h-6 text-white fill-current ml-1" />
                     )}
                  </button>

                  <div className="flex-1">
                     <div className="flex justify-between text-xs text-gray-400 mb-1 font-mono">
                        <span>{formatTime(currentTime)}</span>
                        <span>{formatTime(duration)}</span>
                     </div>
                     {/* Custom Range Input */}
                     <input 
                        type="range"
                        min="0"
                        max={duration || 100}
                        value={currentTime}
                        onChange={handleSeek}
                        className="w-full h-2 bg-gray-700 rounded-lg appearance-none cursor-pointer accent-indigo-500"
                     />
                  </div>
               </div>

               <div className="flex items-center gap-3 text-sm text-gray-500 justify-center">
                  <Music className="w-4 h-4" />
                  <span>Trilha: {selectedTrack?.name}</span>
               </div>

               <audio 
                  ref={audioRef} 
                  src={jingleData.finalMixUrl} 
                  onEnded={() => setIsPlaying(false)}
                  onTimeUpdate={handleTimeUpdate}
                  onLoadedMetadata={handleLoadedMetadata}
                  className="hidden"
                  loop={false}
               />
            </div>

            {/* UP-SELL SECTION */}
            {!isPaid && (
               <div className={`mb-6 p-5 rounded-2xl border transition-all duration-300 relative overflow-hidden ${upsellSelected ? 'bg-gradient-to-br from-gray-800 to-indigo-900/40 border-indigo-500 ring-1 ring-indigo-500' : 'bg-gray-800/50 border-gray-700'}`}>
                  {upsellSelected && <div className="absolute top-0 right-0 p-2 bg-indigo-600 text-xs font-bold text-white rounded-bl-xl">ADICIONADO</div>}
                  
                  <div className="flex flex-col sm:flex-row gap-4 items-center">
                     <div className="w-full sm:w-20 sm:h-28 bg-black rounded-lg flex items-center justify-center relative overflow-hidden flex-shrink-0 shadow-lg border border-gray-700 group">
                        {/* Fake video timeline */}
                        <div className="absolute bottom-2 left-2 right-2 h-1 bg-gray-700 rounded-full overflow-hidden">
                           <div className="w-1/2 h-full bg-indigo-500"></div>
                        </div>
                        <Video className="text-gray-500 w-8 h-8" />
                        <div className="absolute inset-0 bg-gradient-to-t from-black/60 to-transparent" />
                        <Instagram className="absolute top-2 right-2 w-4 h-4 text-white/50" />
                     </div>

                     <div className="flex-1 text-center sm:text-left">
                        <div className="flex items-center justify-center sm:justify-start gap-2 mb-1">
                           <h3 className="font-bold text-lg text-white">Quero o Vídeo para Reels/TikTok</h3>
                           <span className="bg-green-500/20 text-green-400 text-xs px-2 py-0.5 rounded font-bold uppercase">Promoção</span>
                        </div>
                        <p className="text-sm text-gray-400 mb-2">
                           Transformamos esse áudio em um vídeo vertical profissional usando <strong>suas fotos</strong>. Perfeito para postar agora!
                        </p>
                        <p className="text-xs text-gray-500 italic">
                           * Você enviará as fotos após o pagamento.
                        </p>
                     </div>

                     <div className="flex flex-col items-center gap-2 min-w-[120px]">
                        <span className="text-xl font-bold text-white">+ R$ {PRICE_VIDEO_UPSELL.toFixed(2).replace('.', ',')}</span>
                        <button 
                           onClick={() => setUpsellSelected(!upsellSelected)}
                           className={`px-4 py-2 rounded-lg font-semibold text-sm transition-all w-full flex items-center justify-center gap-2 ${
                              upsellSelected 
                              ? 'bg-red-500/10 text-red-400 hover:bg-red-500/20 border border-red-500/30' 
                              : 'bg-indigo-600 hover:bg-indigo-500 text-white shadow-lg'
                           }`}
                        >
                           {upsellSelected ? 'Remover' : 'Adicionar'}
                        </button>
                     </div>
                  </div>
               </div>
            )}

            {/* Action Buttons */}
            <div className="mt-auto">
               {!isPaid ? (
                 <div className="space-y-3">
                    <button 
                       onClick={handleBuyClick}
                       className="w-full flex items-center justify-center gap-2 bg-emerald-600 hover:bg-emerald-500 text-white px-6 py-4 rounded-xl font-bold transition-all shadow-lg hover:shadow-emerald-500/20 active:scale-[0.98] animate-pulse"
                    >
                       <CreditCard className="w-5 h-5" />
                       Comprar e Baixar - R$ {getTotalPrice().toFixed(2).replace('.', ',')}
                    </button>
                    {/* Botão de Exemplo para Cakto / Link Externo */}
                    <button 
                       onClick={handleCaktoRedirect}
                       className="w-full flex items-center justify-center gap-2 bg-gray-700 hover:bg-gray-600 text-white px-6 py-3 rounded-xl font-semibold transition-all text-sm"
                    >
                       <ExternalLink className="w-4 h-4" />
                       Pagar com Cakto (Simulação)
                    </button>
                 </div>
               ) : (
                 <div className="space-y-4">
                     <a 
                        href={jingleData.finalMixUrl} 
                        download={`jingle-${jingleData.trackId}.wav`}
                        className="w-full flex items-center justify-center gap-2 bg-indigo-600 hover:bg-indigo-500 text-white px-6 py-4 rounded-xl font-bold transition-all shadow-lg hover:shadow-indigo-500/20 active:scale-[0.98]"
                     >
                        <Download className="w-5 h-5" /> 
                        Baixar Áudio (WAV)
                     </a>
                     
                     {upsellSelected && (
                        <div className="bg-gray-800 p-4 rounded-xl border border-indigo-500/50 flex flex-col items-center text-center">
                           <CheckCircle2 className="w-8 h-8 text-green-500 mb-2" />
                           <h4 className="font-bold text-white mb-1">Vídeo Confirmado!</h4>
                           <p className="text-sm text-gray-400 mb-3">
                              Para começarmos a edição do seu vídeo no CapCut, clique abaixo e envie as fotos e o comprovante.
                           </p>
                           <a 
                             href={getWhatsAppLink()}
                             target="_blank"
                             rel="noopener noreferrer"
                             className="w-full text-sm bg-green-600 hover:bg-green-500 text-white px-4 py-3 rounded-lg font-bold flex items-center justify-center gap-2 shadow-lg shadow-green-900/30 transition-transform hover:-translate-y-1"
                           >
                              <MessageCircle className="w-5 h-5" />
                              Enviar Fotos no WhatsApp
                           </a>
                        </div>
                     )}
                 </div>
               )}
               
               <p className="text-center text-xs text-gray-500 mt-3">
                  {isPaid ? "Licença comercial vitalícia incluída." : "A prévia contém marca d'água de áudio. Compre para remover."}
               </p>
            </div>

          </div>
        )}

        {/* Payment Modal (Manual) */}
        {showPaymentModal && (
          <div className="fixed inset-0 z-50 flex items-center justify-center p-4 bg-black/80 backdrop-blur-sm animate-fade-in">
             <div className="bg-gray-800 border border-gray-600 rounded-2xl p-6 w-full max-w-md shadow-2xl relative">
                <button 
                   onClick={() => setShowPaymentModal(false)}
                   className="absolute top-4 right-4 text-gray-400 hover:text-white"
                >
                   <X className="w-6 h-6" />
                </button>

                <h3 className="text-2xl font-bold text-white mb-1">Finalizar Compra</h3>
                <p className="text-gray-400 text-sm mb-6">Libere o download imediato.</p>

                {/* Receipt */}
                <div className="bg-gray-900/50 p-4 rounded-xl mb-6 space-y-2 text-sm">
                   <div className="flex justify-between text-gray-300">
                      <span>Locução + Trilha Comercial</span>
                      <span>R$ {PRICE_JINGLE.toFixed(2).replace('.', ',')}</span>
                   </div>
                   {upsellSelected && (
                      <div className="flex justify-between text-indigo-300 font-medium">
                         <span>+ Vídeo Vertical (Reels)</span>
                         <span>R$ {PRICE_VIDEO_UPSELL.toFixed(2).replace('.', ',')}</span>
                      </div>
                   )}
                   <div className="h-px bg-gray-700 my-2"></div>
                   <div className="flex justify-between text-white font-bold text-lg">
                      <span>Total</span>
                      <span>R$ {getTotalPrice().toFixed(2).replace('.', ',')}</span>
                   </div>
                </div>

                <div className="bg-white p-6 rounded-xl flex flex-col items-center mb-6">
                   {/* Placeholder QR Code */}
                   <div className="bg-gray-200 w-48 h-48 rounded mb-4 flex items-center justify-center text-gray-400 border-2 border-dashed border-gray-400">
                      <QrCode className="w-16 h-16 opacity-20" />
                   </div>
                   <p className="text-gray-500 text-xs text-center">Escaneie o QR Code com seu app de banco</p>
                </div>

                <div className="space-y-3">
                   <button className="w-full bg-gray-700 hover:bg-gray-600 text-white py-3 rounded-lg font-medium text-sm transition-colors flex items-center justify-center gap-2">
                      Copiar código PIX
                   </button>
                   <button 
                      onClick={handleConfirmPayment}
                      className="w-full bg-green-600 hover:bg-green-500 text-white py-3 rounded-lg font-bold shadow-lg shadow-green-900/20 transition-all active:scale-95"
                   >
                      Já fiz o pagamento
                   </button>
                </div>

                <p className="text-center text-xs text-gray-500 mt-4 flex items-center justify-center gap-1">
                   <Lock className="w-3 h-3" /> Pagamento 100% seguro via Mercado Pago
                </p>
             </div>
          </div>
        )}

      </main>
      
      <footer className="mt-12 text-center text-gray-600 text-sm">
        <p>Desenvolvido com Google Gemini 2.5 Flash & Flash-TTS</p>
      </footer>
    </div>
  );
}