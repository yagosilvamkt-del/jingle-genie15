export const config = {
  runtime: 'edge', // Usa Edge Runtime para resposta rápida
};

/**
 * Esta função roda no servidor da Vercel.
 * Você deve configurar na Cakto/Gateway a URL de Webhook: https://seu-site.vercel.app/api/webhook
 */
export default async function handler(request: Request) {
  if (request.method !== 'POST') {
    return new Response('Method Not Allowed', { status: 405 });
  }

  try {
    const payload = await request.json();

    // LOG para você ver o que está chegando da Cakto no painel da Vercel
    console.log('Webhook recebido:', payload);

    // 1. Verificação de Segurança (Recomendado)
    // Valide se o token do webhook bate com o que você configurou na Cakto
    // const secret = request.headers.get('x-webhook-secret');
    // if (secret !== process.env.CAKTO_WEBHOOK_SECRET) return new Response('Unauthorized', { status: 401 });

    // 2. Lógica de Aprovação
    // Dependendo da estrutura da Cakto, o status pode vir como 'paid', 'approved', etc.
    const status = payload.status || payload.current_status;

    if (status === 'paid' || status === 'approved') {
      const userEmail = payload.customer?.email;
      const trackId = payload.metadata?.trackId; // Você pode passar isso no checkout

      // AQUI VOCÊ PODE:
      // A) Salvar no banco de dados que este email comprou
      // B) Enviar o arquivo por email (usando Resend, SendGrid, etc)
      
      console.log(`Pagamento aprovado para ${userEmail} - Track: ${trackId}`);
      
      return new Response(JSON.stringify({ received: true, status: 'approved' }), {
        status: 200,
        headers: { 'Content-Type': 'application/json' }
      });
    }

    return new Response(JSON.stringify({ received: true, status: 'pending' }), { status: 200 });

  } catch (error) {
    console.error('Erro no Webhook:', error);
    return new Response('Bad Request', { status: 400 });
  }
}