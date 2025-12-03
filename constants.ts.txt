import { Track, TrackGenre, TrackCategory, Voice } from './types';

export const TRACK_LIBRARY: Track[] = [
  // --- LOJAS ---
  {
    id: 't1',
    name: 'Pop de Verão',
    genre: TrackGenre.UPBEAT,
    category: TrackCategory.STORE,
    description: 'Batida pop energética perfeita para vendas relâmpago e ofertas urgentes.',
    cuePoint: 4.5,
    duration: 40,
    bpm: 120,
    color: 'bg-orange-500',
    suggestedScript: 'Atenção! Somente hoje na [NOME DA LOJA] você encontra ofertas imperdíveis com 50% de desconto. Corra e aproveite!'
  },
  {
    id: 't2',
    name: 'Corporativo Suave',
    genre: TrackGenre.CORPORATE,
    category: TrackCategory.STORE,
    description: 'Fundo suave e confiável para anúncios de serviços e empresas.',
    cuePoint: 2.0,
    duration: 40,
    bpm: 90,
    color: 'bg-blue-500',
    suggestedScript: 'Na [NOME DA EMPRESA], nós cuidamos do que é importante para você. Conheça nossos novos planos de serviço.'
  },
  {
    id: 't5',
    name: 'O Natal Está Chegando',
    genre: TrackGenre.CHRISTMAS,
    category: TrackCategory.STORE,
    description: 'Trilha natalina envolvente. Voz entra aos 11s.',
    cuePoint: 11.0,
    duration: 40,
    bpm: 95,
    color: 'bg-emerald-600',
    suggestedScript: 'Aqui na (NOME DA LOJA) o natal já começou e nossa loja está repleta de muitas promoções e novidades!',
    fileUrl: '/Nat.mp3'
  },
  {
    id: 't6',
    name: 'Véspera de Natal',
    genre: TrackGenre.CHRISTMAS,
    category: TrackCategory.STORE,
    description: 'Clássico tema natalino para mensagens emocionantes.',
    cuePoint: 3.5,
    duration: 40,
    bpm: 100,
    color: 'bg-red-600',
    suggestedScript: 'Neste Natal, celebre o amor e a alegria com quem você ama. Aproveite nossas ofertas especiais.',
  },
  {
    id: 't3',
    name: 'Magia do Feriado',
    genre: TrackGenre.CALM,
    category: TrackCategory.STORE,
    description: 'Sinos e pads acolhedores para datas especiais.',
    cuePoint: 5.0,
    duration: 40,
    bpm: 80,
    color: 'bg-red-500',
  },

  // --- CANDIDATOS ---
  {
    id: 't4',
    name: 'Lançamento Épico',
    genre: TrackGenre.DRAMATIC,
    category: TrackCategory.CANDIDATE,
    description: 'Bateria cinematográfica para grandes discursos de mudança.',
    cuePoint: 6.0,
    duration: 40,
    bpm: 110,
    color: 'bg-purple-600',
    suggestedScript: 'Chegou a hora da mudança. Com coragem e determinação, vamos construir um futuro melhor para nossa cidade. Vote [NÚMERO]!'
  },
  {
    id: 't7',
    name: 'Jornada de Esperança',
    genre: TrackGenre.POLITICAL,
    category: TrackCategory.CANDIDATE,
    description: 'Orquestra inspiradora e motivacional. Perfeita para apresentar propostas.',
    cuePoint: 3.0,
    duration: 40,
    bpm: 110,
    color: 'bg-yellow-600',
    suggestedScript: 'Eu sou [NOME], e meu compromisso é com você. Juntos, vamos renovar a saúde e a educação. A esperança venceu o medo.'
  },
  {
    id: 't8',
    name: 'Vitória Certa',
    genre: TrackGenre.UPBEAT,
    category: TrackCategory.CANDIDATE,
    description: 'Ritmo alegre e popular para jingles de campanha de rua.',
    cuePoint: 2.0,
    duration: 40,
    bpm: 125,
    color: 'bg-blue-600',
    suggestedScript: 'É o povo na rua, é a vitória da gente! Para vereador, vote no amigo da comunidade. Vote [NÚMERO]!'
  }
];

export const VOICE_LIBRARY: Voice[] = [
  { 
    id: 'Puck', 
    name: 'Puck', 
    gender: 'Masculino', 
    style: 'Suave', 
    description: 'Tom neutro e amigável, ótimo para narrações gerais.' 
  },
  { 
    id: 'Charon', 
    name: 'Charon', 
    gender: 'Masculino', 
    style: 'Profundo', 
    description: 'Voz grave e autoritária para anúncios sérios.' 
  },
  { 
    id: 'Kore', 
    name: 'Kore', 
    gender: 'Feminino', 
    style: 'Calmo', 
    description: 'Voz relaxante e serena para temas delicados.' 
  },
  { 
    id: 'Fenrir', 
    name: 'Fenrir', 
    gender: 'Masculino', 
    style: 'Intenso', 
    description: 'Voz forte e energética para promoções de impacto.' 
  },
  { 
    id: 'Zephyr', 
    name: 'Zephyr', 
    gender: 'Feminino', 
    style: 'Energético', 
    description: 'Voz confiante e ágil, perfeita para varejo.' 
  },
];

export const SCRIPT_TEMPLATES = [
  {
    id: 'offer',
    label: '🔥 Tem Oferta',
    text: 'Atenção! O patrão ficou maluco! Somente neste fim de semana, toda a linha de [PRODUTO] com 50% de desconto. É isso mesmo, metade do preço! Corra para a [NOME DA LOJA] antes que acabe o estoque.'
  },
  {
    id: 'xmas',
    label: '🎄 Natal',
    text: 'Neste Natal, o presente perfeito está na [NOME DA LOJA]. Preparamos ofertas mágicas para você encantar quem você ama. Venha conferir nossa coleção especial e parcele em até 10 vezes sem juros. Feliz Natal!'
  },
  {
    id: 'newyear',
    label: '✨ Ano Novo',
    text: '3, 2, 1... Feliz Ano Novo! Comece o ano com o pé direito e economizando muito. A [NOME DA LOJA] deseja a todos os clientes um ano repleto de conquistas, saúde e muitas promoções. Venha brindar com a gente!'
  },
  {
    id: 'blackfriday',
    label: '🖤 Black Friday',
    text: 'Está preparada? Chegou a Black Friday da [NOME DA LOJA]! Preços nunca vistos, descontos reais e condições imperdíveis. Não compre nada agora, espere pela nossa Black Friday. É a oportunidade do ano!'
  },
  {
    id: 'candidate',
    label: '🤝 Candidato',
    text: 'Para mudar de verdade, a gente precisa de coragem e trabalho sério. Eu sou [SEU NOME], candidato a [CARGO]. Meu compromisso é com a saúde, a educação e com você. No dia da eleição, vote certo, vote [NÚMERO].'
  },
  {
    id: 'random',
    label: '🎲 Aleatório',
    text: ''
  }
];