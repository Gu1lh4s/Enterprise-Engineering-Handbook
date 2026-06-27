# LGPD — Guia de Implementação para Engenheiros

> **Category:** Legal / Compliance
> **Version:** 1.0.0
> **Regulation:** Lei nº 13.709/2018 — Lei Geral de Proteção de Dados Pessoais (Brazil)
> **Effective:** September 2020 (sanctions from August 2021)
> **Regulator:** ANPD — Autoridade Nacional de Proteção de Dados
> **Penalties:** Up to BRL 50M per violation or 2% of Brazilian revenue

---

## Table of Contents

1. [Visão Geral e Comparação com GDPR](#1-visão-geral-e-comparação-com-gdpr)
2. [Conceitos Fundamentais](#2-conceitos-fundamentais)
3. [Bases Legais para Tratamento](#3-bases-legais-para-tratamento)
4. [Direitos do Titular](#4-direitos-do-titular)
5. [Privacy by Design](#5-privacy-by-design)
6. [Relatório de Impacto (RIPD)](#6-relatório-de-impacto-ripd)
7. [Tratamento de Dados de Crianças](#7-tratamento-de-dados-de-crianças)
8. [Segurança e Incidentes](#8-segurança-e-incidentes)
9. [Encarregado (DPO)](#9-encarregado-dpo)
10. [Implementação Técnica](#10-implementação-técnica)
11. [Checklist LGPD](#11-checklist-lgpd)
12. [Referências](#12-referências)

---

## 1. Visão Geral e Comparação com GDPR

A LGPD foi fortemente inspirada pela GDPR europeia, mas com diferenças importantes:

| Aspecto | GDPR | LGPD |
|---|---|---|
| Jurisdição | União Europeia | Brasil |
| Publicação | Maio 2018 | Agosto 2018 |
| Vigência | Maio 2018 | Setembro 2020 |
| Sanções | €20M ou 4% do faturamento global | BRL 50M ou 2% do faturamento no Brasil |
| Regulator | Autoridade supervisora de cada estado membro | ANPD (federal) |
| Bases legais | 6 | 10 |
| Encarregado (DPO) | Obrigatório em muitos casos | Obrigatório sempre |
| Transferência internacional | Mecanismo explícito | Critérios semelhantes |
| Dados sensíveis | Categorias especiais (9) | Dados sensíveis (10) |

### Escopo Extraterritorial

A LGPD aplica-se quando:
```
a) O tratamento ocorre no Brasil
b) A atividade de tratamento destina-se a oferecer bens/serviços a pessoas no Brasil
c) Os dados foram coletados no território brasileiro

Não importa onde a empresa está sediada.
Empresa americana com usuários brasileiros → deve cumprir LGPD
```

---

## 2. Conceitos Fundamentais

### Definições (Art. 5º LGPD)

| Termo | Definição | Exemplo técnico |
|---|---|---|
| **Dado pessoal** | Informação que identifica ou pode identificar pessoa natural | Email, CPF, IP, cookie ID |
| **Dado pessoal sensível** | Origem racial, dado genético, biométrico, saúde, vida sexual, religião, opinião política, filiação sindical | Retina scan, prontuário médico |
| **Titular** | Pessoa natural a quem os dados se referem | Seus usuários |
| **Controlador** | Quem decide sobre o tratamento | Sua empresa |
| **Operador** | Quem trata dados em nome do controlador | AWS, Stripe, Supabase |
| **Encarregado** | Pessoa indicada pelo controlador (DPO) | Seu Privacy Officer |
| **Tratamento** | Toda operação sobre dados pessoais | Coleta, armazenamento, uso, compartilhamento, eliminação |
| **Consentimento** | Manifestação livre, informada e inequívoca | Opt-in por checkbox |
| **Anonimização** | Impossibilidade de identificação por meios razoáveis | Hashing irreversível + remoção de identificadores diretos |
| **Pseudonimização** | Tratamento com identificadores não diretamente associados ao titular | Token interno que requer tabela de mapeamento para re-identificar |

### Dados Sensíveis (Art. 5º, II)

Tratamento de dados sensíveis exige **consentimento específico e destacado** (ou exceção expressa):

```
1. Origem racial ou étnica
2. Convicção religiosa
3. Opinião política
4. Filiação a sindicato ou organização de caráter religioso, filosófico ou político
5. Dados referentes à saúde ou à vida sexual
6. Dados genéticos ou biométricos (quando vinculados a pessoa natural)

Para dados sensíveis de saúde — base legal adicional:
- Necessidade de atenção à saúde
- Garantia de prevenção à fraude e à segurança do titular
- Proteção da vida ou da incolumidade física do titular
- Exercício regular de direitos
```

---

## 3. Bases Legais para Tratamento

A LGPD tem **10 bases legais** (vs 6 do GDPR):

| Base Legal | Art. | Quando Usar | Exemplo |
|---|---|---|---|
| **Consentimento** | 7º, I | Opt-in para marketing, cookies | Newsletter, remarketing |
| **Cumprimento de obrigação legal** | 7º, II | Exigência por lei | Emissão de NF, retenção fiscal |
| **Execução de políticas públicas** | 7º, III | Entes públicos | — |
| **Estudos por órgão de pesquisa** | 7º, IV | Pesquisa científica | Dados anonimizados para pesquisa |
| **Execução de contrato** | 7º, V | Necessário para cumprir contrato | Enviar produto ao endereço fornecido |
| **Exercício regular de direitos** | 7º, VI | Defesa em processo judicial/administrativo | Guardar logs para contencioso |
| **Proteção da vida** | 7º, VII | Emergência médica | — |
| **Tutela da saúde** | 7º, VIII | Profissionais de saúde | Prontuário eletrônico |
| **Legítimo interesse** | 7º, IX | Interesse legítimo do controlador/terceiros | Prevenção de fraude, segurança de rede |
| **Proteção ao crédito** | 7º, X | Análise de crédito | Score de crédito |

### Consentimento LGPD

Requisitos mais rígidos que o GDPR em alguns aspectos:

```sql
-- Tabela de registros de consentimento (RoC)
CREATE TABLE consent_records_lgpd (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  titular_id      UUID REFERENCES users(id),
  finalidade      TEXT NOT NULL,        -- Propósito específico
  base_legal      TEXT NOT NULL,        -- Base legal utilizada
  consentimento   BOOLEAN,              -- True = dado, False = negado/revogado
  data_coleta     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  data_revogacao  TIMESTAMPTZ,
  ip_address      INET,
  user_agent      TEXT,
  texto_exibido   TEXT,                 -- Texto exato mostrado ao titular
  versao_politica TEXT,                 -- Versão da política de privacidade
  canal           TEXT                  -- 'site_signup', 'app', 'termo_papel'
);

-- LGPD Art. 8º §5: ônus da prova é do controlador
-- Controlador DEVE conseguir provar que o consentimento foi dado
```

---

## 4. Direitos do Titular

LGPD Art. 18 — direitos análogos ao GDPR mas com especificidades:

| Direito | Art. | Prazo | Implementação |
|---|---|---|---|
| Confirmação de tratamento | 18, I | 15 dias¹ | Endpoint de verificação |
| Acesso aos dados | 18, II | 15 dias¹ | Exportação de dados |
| Correção de dados incompletos/inexatos | 18, III | 15 dias¹ | Interface de edição |
| Anonimização, bloqueio ou eliminação | 18, IV | Razoável | Fluxo de exclusão |
| Portabilidade | 18, V | A definir pela ANPD | Export JSON/CSV |
| Informação sobre compartilhamento | 18, VI | 15 dias¹ | Lista de operadores |
| Informação sobre não consentimento | 18, VII | 15 dias¹ | Aviso de consequências |
| Revogação do consentimento | 18, IX | Imediato | Opt-out instantâneo |
| Revisão de decisão automatizada | 20 | 15 dias¹ | Human review |
| Peticionar à ANPD | 18, § 1º | — | Contato da ANPD |

¹ ANPD não definiu prazo expresso — referência é regulamentação posterior. Use 15 dias como prática recomendada.

### Implementação: Endpoint de Direitos

```typescript
// Router para direitos do titular
app.post('/api/privacidade/solicitacao', authenticate, async (req, res) => {
  const { tipo, detalhes } = req.body
  const titularId = req.user.id

  type TipoSolicitacao = 
    | 'confirmacao_tratamento'
    | 'acesso_dados'
    | 'correcao'
    | 'eliminacao'
    | 'portabilidade'
    | 'informacao_compartilhamento'
    | 'revogacao_consentimento'
    | 'revisao_decisao_automatizada'

  if (!Object.values(TIPOS_VALIDOS).includes(tipo)) {
    return res.status(400).json({ erro: 'Tipo de solicitação inválido' })
  }

  // Registrar solicitação
  const solicitacao = await db.query(`
    INSERT INTO solicitacoes_lgpd (titular_id, tipo, detalhes, status, criado_em)
    VALUES ($1, $2, $3, 'pendente', NOW())
    RETURNING id
  `, [titularId, tipo, detalhes])

  // Enviar email de confirmação de recebimento
  await enviarEmail({
    destinatario: req.user.email,
    assunto: 'Recebemos sua solicitação de privacidade',
    corpo: `Protocolo: ${solicitacao.rows[0].id}. Responderemos em até 15 dias.`,
  })

  // Processar automaticamente o que for possível
  if (tipo === 'revogacao_consentimento') {
    await revogarConsentimento(titularId, detalhes.finalidade)
  }

  if (tipo === 'acesso_dados') {
    await gerarExportacaoDados(titularId, solicitacao.rows[0].id)
  }

  res.json({
    protocolo: solicitacao.rows[0].id,
    tipo,
    prazo: '15 dias úteis',
    status: 'recebido',
  })
})
```

---

## 5. Privacy by Design

Art. 46 LGPD — medidas técnicas e administrativas devem ser adotadas desde a concepção:

```typescript
// Exemplo: nova feature de agenda — privacy by design

// ❌ Design sem privacidade
interface AgendaItem {
  userId: string
  clientEmail: string       // Compartilhado com todos os funcionários
  clientPhone: string       // Visível na listagem geral
  clientCpf: string         // Armazenado em plaintext no frontend
  notes: string             // Notas médicas visíveis para todos
  serviceType: string
  time: Date
}

// ✅ Design com privacidade
interface AgendaItem {
  id: string
  serviceType: string       // Não-pessoal: pode ser compartilhado
  time: Date
  status: string
  // Dados pessoais acessíveis apenas via API com autorização específica
  // Funcionário vê: nome do cliente (apenas)
  // Gerente vê: nome + contato
  // Admin vê: tudo
}

interface ClienteAgenda {
  nomeExibicao: string      // "Maria S." — não o nome completo por padrão
  // Demais dados acessíveis via endpoint separado com permissão
}
```

---

## 6. Relatório de Impacto (RIPD)

Art. 38 LGPD — Autoridade Nacional pode determinar ao controlador que elabore RIPD para atividades de tratamento que possam gerar riscos.

**Diferença GDPR x LGPD:**
- GDPR DPIA: exigida proativamente para alto risco
- LGPD RIPD: pode ser exigida pela ANPD; controlador decide quando fazer

**Recomendação:** Adotar proativamente para:
```
- Tratamento em larga escala de dados pessoais
- Dados sensíveis
- Perfis comportamentais
- Decisões automatizadas com efeitos jurídicos
- Novas tecnologias (IA, biometria, IoT)
- Vigilância sistemática
```

**Conteúdo mínimo do RIPD:**
```markdown
# RIPD — [Nome do Tratamento]

## 1. Descrição do Tratamento
- Propósito e necessidade
- Tipos de dados e titulares
- Ciclo de vida dos dados

## 2. Agentes de Tratamento
- Controlador e contatos
- Operadores envolvidos
- Encarregado (DPO)

## 3. Medidas de Segurança
- Controles técnicos implementados
- Avaliação de riscos residuais

## 4. Riscos Identificados
- Para cada risco: probabilidade × impacto → nível de risco
- Medidas de mitigação

## 5. Medidas de Mitigação
- Controles adicionais previstos
- Cronograma de implementação
```

---

## 7. Tratamento de Dados de Crianças

Art. 14 LGPD — dados de crianças (menores de 12 anos):
```
- Apenas com consentimento específico de um dos pais ou responsável legal
- Nenhum dado de criança sem verificação de consentimento parental
- Vedado o tratamento para finalidade que vise lucro com dados de criança

Adolescentes (12-18 anos): aplicam-se as demais bases legais, mas com cuidado especial
```

```typescript
// Verificação de idade na coleta
async function validarIdade(dataNascimento: string, email: string): Promise<void> {
  const idade = calcularIdade(new Date(dataNascimento))
  
  if (idade < 12) {
    throw new Error('LGPD Art. 14: Tratamento de dados de crianças requer consentimento parental específico')
  }
  
  if (idade < 18) {
    // Log para auditoria — adolescente
    await logPrivacidade({ tipo: 'ADOLESCENTE_CADASTRO', email, idade })
  }
}

// Fluxo de consentimento parental
async function solicitarConsentimentoParental(
  criancaEmail: string,
  responsavelEmail: string,
  finalidade: string
): Promise<string> {
  const token = crypto.randomBytes(32).toString('hex')
  
  await db.query(`
    INSERT INTO consentimento_parental (
      crianca_email, responsavel_email, finalidade,
      token, status, criado_em, expira_em
    ) VALUES ($1, $2, $3, $4, 'pendente', NOW(), NOW() + INTERVAL '7 days')
  `, [criancaEmail, responsavelEmail, finalidade, token])
  
  await enviarEmail({
    destinatario: responsavelEmail,
    assunto: 'Autorização para cadastro de menor de idade',
    corpo: `Acesse o link para autorizar: ${BASE_URL}/consentimento-parental/${token}`,
  })
  
  return token
}
```

---

## 8. Segurança e Incidentes

### Requisitos Técnicos (Art. 46)

```
Medidas técnicas e administrativas aptas a proteger os dados pessoais:
- De acessos não autorizados
- De situações acidentais ou ilícitas de destruição, perda, alteração, comunicação
- De qualquer forma de tratamento inadequado ou ilícito

ANPD pode dispor sobre padrões técnicos mínimos (ainda em construção)
```

### Comunicação de Incidentes (Art. 48)

```
Prazo: "em prazo razoável" — ANPD definirá (orientação atual: 2 dias úteis)

Comunicar à ANPD e aos titulares afetados quando:
- Incidente de segurança que possa acarretar risco ou dano relevante aos titulares

Informações obrigatórias na comunicação:
1. Descrição da natureza dos dados afetados
2. Informações sobre os titulares envolvidos
3. Indicação das medidas técnicas e de segurança adotadas
4. Riscos relacionados ao incidente
5. Medidas adotadas para reverter ou mitigar os efeitos do incidente
```

```typescript
// Playbook de incidente LGPD
interface IncidenteLGPD {
  id: string
  descoberto_em: Date
  descricao: string
  dados_afetados: string[]      // Tipos de dados: email, CPF, etc.
  titulares_afetados: number    // Estimativa
  causa_raiz?: string
  medidas_contencao: string[]
  risco_nivel: 'baixo' | 'medio' | 'alto' | 'critico'
  comunicacao_anpd?: Date
  comunicacao_titulares?: Date
}

async function registrarIncidente(incidente: Omit<IncidenteLGPD, 'id'>): Promise<string> {
  const id = crypto.randomUUID()
  
  await db.query(`
    INSERT INTO incidentes_lgpd (id, descoberto_em, descricao, dados_afetados, 
      titulares_afetados, risco_nivel, criado_em)
    VALUES ($1, $2, $3, $4, $5, $6, NOW())
  `, [id, incidente.descoberto_em, incidente.descricao, 
      JSON.stringify(incidente.dados_afetados),
      incidente.titulares_afetados, incidente.risco_nivel])
  
  // Alertar encarregado (DPO) imediatamente
  await alertarEncarregado({ incidenteId: id, ...incidente })
  
  // Se risco alto ou crítico: iniciar processo de notificação ANPD
  if (['alto', 'critico'].includes(incidente.risco_nivel)) {
    await agendarNotificacaoANPD(id)  // Deve ocorrer em até 2 dias úteis
  }
  
  return id
}
```

---

## 9. Encarregado (DPO)

Art. 41 LGPD — todo controlador deve indicar encarregado:

```
Atribuições do Encarregado:
- Aceitar reclamações e comunicações dos titulares
- Prestar esclarecimentos e adotar providências
- Receber comunicações da ANPD e adotar providências
- Orientar funcionários e contratados sobre práticas de proteção de dados
- Executar as demais atribuições determinadas pelo controlador
- Atuar como canal de comunicação entre controlador, titulares e ANPD

Identidade e dados de contato do encarregado:
- Devem ser divulgados publicamente na política de privacidade
- Podem ser: funcionário interno ou empresa terceirizada (DPO-as-a-Service)

ANPD pode dispensar a obrigação para:
- Empresas de pequeno porte
- Startups (critérios ainda sendo definidos)
```

---

## 10. Implementação Técnica

### Mapeamento de Dados Pessoais

```typescript
// Inventário de dados — base para toda a conformidade
const MAPEAMENTO_DADOS = {
  tabela: 'users',
  campos: {
    email:          { sensivel: false, base_legal: 'contrato', retencao: '2 anos após encerramento' },
    nome:           { sensivel: false, base_legal: 'contrato', retencao: '2 anos após encerramento' },
    cpf:            { sensivel: false, base_legal: 'obrigacao_legal', retencao: '5 anos', criptografar: true },
    data_nascimento:{ sensivel: false, base_legal: 'contrato', retencao: '2 anos após encerramento' },
    telefone:       { sensivel: false, base_legal: 'contrato', retencao: '2 anos após encerramento' },
    ip_cadastro:    { sensivel: false, base_legal: 'legitimo_interesse', retencao: '90 dias' },
  }
}

const OPERADORES = [
  { nome: 'AWS', finalidade: 'Hospedagem', pais: 'EUA', dpa_assinado: true, mecanismo_transferencia: 'SCCs' },
  { nome: 'Supabase', finalidade: 'Banco de dados', pais: 'EUA', dpa_assinado: true, mecanismo_transferencia: 'SCCs' },
  { nome: 'Stripe', finalidade: 'Pagamentos', pais: 'EUA', dpa_assinado: true, mecanismo_transferencia: 'SCCs' },
  { nome: 'SendGrid', finalidade: 'Email transacional', pais: 'EUA', dpa_assinado: true, mecanismo_transferencia: 'SCCs' },
]
```

### Política de Privacidade — Elementos Obrigatórios

```markdown
A política de privacidade LGPD deve conter (Art. 9º):

1. Finalidade específica do tratamento
2. Forma e duração do tratamento
3. Identificação do controlador
4. Informações de contato do controlador
5. Contato do encarregado de dados
6. Informações sobre compartilhamento de dados por operadores
7. Responsabilidades dos agentes de tratamento
8. Direitos do titular (Art. 18) com procedimento de exercício
9. Bases legais para cada finalidade de tratamento
10. Informações sobre transferências internacionais
```

### Transferências Internacionais (Art. 33)

```
Dados pessoais podem ser transferidos para outros países se:
a) País com nível adequado de proteção (decisão de adequação da ANPD)
b) Controlador oferecer garantias: cláusulas contratuais, normas corporativas globais, selos/certificações
c) Consentimento específico (não recomendado como única base)
d) Necessário para execução de contrato
e) Cooperação jurídica internacional
f) Proteção da vida do titular
g) Cumprimento de obrigação legal
h) Exercício de direitos em processo judicial
```

---

## 11. Checklist LGPD

**Base Legal e Documentação**
- [ ] Base legal identificada para cada finalidade de tratamento
- [ ] Mapeamento de dados pessoais (inventário completo)
- [ ] Política de privacidade publicada e acessível
- [ ] Encarregado nomeado e contato publicado
- [ ] DPA assinado com todos os operadores

**Consentimento**
- [ ] Consentimento registrado com data, IP, texto exibido, versão da política
- [ ] Opt-in ativo (sem pré-marcação)
- [ ] Mecanismo de revogação acessível
- [ ] Consentimento parental para crianças (< 12 anos)

**Direitos do Titular**
- [ ] Portal ou processo para exercício de direitos
- [ ] SLA de 15 dias documentado
- [ ] Fluxo de eliminação implementado e testado
- [ ] Portabilidade: export em formato estruturado (JSON/CSV)
- [ ] Registro de todas as solicitações recebidas

**Segurança Técnica**
- [ ] Criptografia em trânsito (TLS 1.2+)
- [ ] Criptografia em repouso para dados sensíveis
- [ ] Controle de acesso com mínimo privilégio
- [ ] Logs de acesso a dados pessoais
- [ ] Política de retenção com eliminação automática

**Incidentes**
- [ ] Processo de detecção de incidentes
- [ ] Playbook de resposta documentado
- [ ] Processo de notificação à ANPD (2 dias úteis)
- [ ] Processo de notificação aos titulares

**Transferência Internacional**
- [ ] Países e mecanismo legal documentados
- [ ] SCCs em vigor para EUA, UE e outros
- [ ] Cláusulas de proteção em contratos com operadores internacionais

---

## 12. Referências

- [Texto Integral da LGPD — L13709/2018](https://www.planalto.gov.br/ccivil_03/_ato2015-2018/2018/lei/l13709.htm)
- [ANPD — Autoridade Nacional de Proteção de Dados](https://www.gov.br/anpd)
- [Guia Orientativo da ANPD](https://www.gov.br/anpd/pt-br/documentos-e-publicacoes/guias-e-manuais)
- [Resolução CD/ANPD nº 2/2022 — Pequenas Empresas](https://www.gov.br/anpd/pt-br/documentos-e-publicacoes/resolucoes)
- [Comparativo LGPD x GDPR — IAPP](https://iapp.org/resources/article/brazil-lgpd-compared-to-europes-gdpr/)
