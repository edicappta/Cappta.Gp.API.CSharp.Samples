<h1>Integração C#</h1>

A Dll da Cappta foi desenvolvida utilizando as melhores práticas de programação e desenvolvimento de software. Utilizamos o padrão COM pensando justamente na integração entre aplicações construídas em várias linguagens. 

Obs: Durante a instalação do CapptaGpPlus o mesmo encarrega-se de registrar a DLL em seu computador.

<h3>Primeira etapa para integração.</h3></br>

Tempo estimado de 01:00 hora

 A primeira etapa consiste na importação do componente (dll) para dentro do projeto.</br>
 No Visual Studio, abra a Solution Explorer, vá em referências e adicione a DLL.
 
	
A primeira função a ser utilizada é **AutenticarPdv()**.</br>
     
Para autenticar é necessário os seguintes dados: CNPJ, PDV e chave de autenticação, estes dados são os mesmos fornecidos durante a instalação do GP.</br>
	
Chave: 795180024C04479982560F61B3C2C06E </br>

OBS: aqui utilizamos um xml para guardar os dados de autenticação.

Em App.config:
```javascript
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <startup>
    <supportedRuntime version="v4.0" sku=".NETFramework,Version=v4.0" />
  </startup>
  <appSettings>
    <add key="ChaveAutenticacao" value="1B489E726C284CC78DE715C7399114BF" />
    <add key="Cnpj" value="34555898000186" />
    <add key="Pdv" value="6" />
    <add key="ClientSettingsProvider.ServiceUri" value="" />
  </appSettings>
  <system.web>
    <membership defaultProvider="ClientAuthenticationMembershipProvider">
      <providers>
        <add name="ClientAuthenticationMembershipProvider" type="System.Web.ClientServices.Providers.ClientFormsAuthenticationMembershipProvider, System.Web.Extensions, Version=4.0.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35" serviceUri="" />
      </providers>
    </membership>
    <roleManager defaultProvider="ClientRoleProvider" enabled="true">
      <providers>
        <add name="ClientRoleProvider" type="System.Web.ClientServices.Providers.ClientRoleProvider, System.Web.Extensions, Version=4.0.0.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35" serviceUri="" cacheTimeout="86400" />
      </providers>
    </roleManager>
  </system.web>
</configuration>
```
Aconselhamos sempre deixar visivel e de fácil acesso para o usuário onde configurar o sistema para utilização do CapptaGpPlus.

```javascript

   private void AutenticarPdv()
   {
	var chaveAutenticacao = ConfigurationManager.AppSettings["ChaveAutenticacao"];
	if (String.IsNullOrWhiteSpace(chaveAutenticacao)) { this.InvalidarAutenticacao("Chave de Autenticação inválida"); }

	var cnpj = ConfigurationManager.AppSettings["Cnpj"];
	if (String.IsNullOrWhiteSpace(cnpj) || cnpj.Length != 14) { this.InvalidarAutenticacao("CNPJ inválido"); }

	int pdv;
	if (Int32.TryParse(ConfigurationManager.AppSettings["Pdv"], out pdv) == false || pdv == 0)
	{
		this.InvalidarAutenticacao("PDV inválido");
	}

	int resultadoAutenticacao = this.cliente.AutenticarPdv(cnpj, pdv, chaveAutenticacao);
	if (resultadoAutenticacao == 0) { return; }

	String mensagem = Mensagens.ResourceManager.GetString(String.Format("RESULTADO_CAPPTA_{0}", resultadoAutenticacao));
		this.ExibeMensagemAutenticacaoInvalida(resultadoAutenticacao);
}
```
O resultado para autenticação com sucesso é: 0

<h1>Primeiro esforço.</h1>
	Toda vez que realizar uma ação com o GP, vai perceber que ele começa a exibir o código 2 para autenticação, não se preocupe é assim mesmo, para recuperar os estados do GP, vamos direto para a etapa 3.

<h1> Etapa 2 </h1>

Tempo estimado de 00:40 minutos

Temos duas formas de integração, a visivel, onde a interação com o usuário fica por conta da Cappta, e a invisivel onde o form pode ser personalizado.


<h3>Para configurar o modo de integração</h3>

```javascript
 private void ConfigurarModoIntegracao(bool exibirInterface)
{
	IConfiguracoes configs = new Configuracoes
	{
		ExibirInterface = exibirInterface
	};

	int resultado = cliente.Configurar(configs);
	if (resultado != 0) { this.CriarMensagemErroPainel(resultado); return; }
}
```

As mensagens de exceção, ficam em um Resources File:

RESULTADO_CAPPTA_1	Não autorizado. Por favor, realize a autenticação para utilizar o CapptaGpPlus
RESULTADO_CAPPTA_10	Uma reimpressão ou cancelamento foi executada dentro de uma sessão multi-cartões
RESULTADO_CAPPTA_2	O CapptaGpPlus esta sendo inicializado, tente novamente em alguns instantes
RESULTADO_CAPPTA_3	O formato da requisição recebida pelo CapptaGpPlus é inválido
RESULTADO_CAPPTA_4	Operação cancelada pelo operador
RESULTADO_CAPPTA_5	Pagamento não autorizado/pendente/não encontrado
RESULTADO_CAPPTA_6	Pagamento ou cancelamento negados pela rede adquirente
RESULTADO_CAPPTA_7	Ocorreu um erro interno no CapptaGpPlus
RESULTADO_CAPPTA_8	Ocorreu um erro na comunicação entre a CappAPI e o CapptaGpPlus
RESULTADO_CAPPTA_9	Não é possível realizar uma operação sem que se tenha finalizado o último pagamento

<h1>Etapa 3</h1>

Tempo estimado de 01:00 hora

Conforme mencionado acima a Iteração Tef é muito importante para o perfeito funcionamento da integração, toda as ações de venda e administrativas passam por esta função. 

```javascript
public void IterarOperacaoTef()
		{
			if (this.RadioButtonUsarMultiTef.Enabled) { this.DesabilitarControlesMultiTef(); }
			this.DesabilitarBotoes();
			IIteracaoTef iteracaoTef = null;

			do
			{
				iteracaoTef = cliente.IterarOperacaoTef();

				if (iteracaoTef is IMensagem)
				{
					this.ExibirMensagem((IMensagem)iteracaoTef);
					Thread.Sleep(INTERVALO_MILISEGUNDOS);
				}

				if (iteracaoTef is IRequisicaoParametro) { this.RequisitarParametros((IRequisicaoParametro)iteracaoTef); }
				if (iteracaoTef is IRespostaTransacaoPendente) { this.ResolverTransacaoPendente((IRespostaTransacaoPendente)iteracaoTef); }

				if (iteracaoTef is IRespostaOperacaoRecusada) { this.ExibirDadosOperacaoRecusada((IRespostaOperacaoRecusada)iteracaoTef); }
				if (iteracaoTef is IRespostaOperacaoAprovada)
				{
					this.ExibirDadosOperacaoAprovada((IRespostaOperacaoAprovada)iteracaoTef);
					this.FinalizarPagamento();
				}

				if (iteracaoTef is IRespostaRecarga)
				{
					this.ExibirDadosDeRecarga((IRespostaRecarga)iteracaoTef);
				}

			} while (this.OperacaoNaoFinalizada(iteracaoTef));

			if (this.sessaoMultiTefEmAndamento == false) { this.HabilitarControlesMultiTef(); }
			this.HabilitarBotoes();
		}

```

Dentro de IterarOperacaoTef() temos alguns métodos:

<h3>Requisitar Parametros</h3>


```javascript
private void RequisitarParametros(IRequisicaoParametro requisicaoParametros)
	{
		string input = Microsoft.VisualBasic.Interaction.InputBox(requisicaoParametros.Mensagem + Environment.NewLine + Environment.NewLine);
		this.cliente.EnviarParametro(input, String.IsNullOrWhiteSpace(input) ? 2 : 1);
		}

```


<h3>Resolver Transacao Pendente</h3>

```javascript
string input = Microsoft.VisualBasic.Interaction.InputBox(requisicaoParametros.Mensagem + Environment.NewLine + Environment.NewLine);
	this.cliente.EnviarParametro(input, String.IsNullOrWhiteSpace(input) ? 2 : 1);
```
<h3>Exibir Dados Operacao Aprovada</h3>

```javascript

StringBuilder mensagemAprovada = new StringBuilder();

	if (resposta.CupomCliente != null) { mensagemAprovada.Append(resposta.CupomCliente.Replace("\"", String.Empty)).AppendLine().AppendLine(); }
	if (resposta.CupomLojista != null) { mensagemAprovada.Append(resposta.CupomLojista.Replace("\"", String.Empty)).AppendLine(); }
	if (resposta.CupomReduzido != null) { mensagemAprovada.Append(resposta.CupomReduzido.Replace("\"", String.Empty)).AppendLine(); }

	this.AtualizarResultado(mensagemAprovada.ToString());
```

<h3>Finalizar Pagamento</h3>

```javascript
if (this.processandoPagamento == false) { return; }

	if (this.sessaoMultiTefEmAndamento)
	{
		quantidadeCartoes--;
		if (this.quantidadeCartoes > 0) { return; }
};

string mensagem = this.GerarMensagemTransacaoAprovada();

this.processandoPagamento = false;
this.sessaoMultiTefEmAndamento = false;

DialogResult result = MessageBox.Show(mensagem.ToString(), "Sample API COM", MessageBoxButtons.OKCancel, MessageBoxIcon.Information, MessageBoxDefaultButton.Button1);
if (result == System.Windows.Forms.DialogResult.OK) { this.cliente.ConfirmarPagamentos(); }
	else { this.cliente.DesfazerPagamentos(); }
```

<h1>Etapa 4</h1>

Tempo estimado de 01:00 hora

Parabéns agora falta pouco, lembrando que a qualquer momento você pode entrar em contato com a equipe tecnica.

Tel: (11) 4302-6179.

Por se tratar de um ambiente de testes, pode ser utilizado cartões reais para as transações, não sera cobrado nada em sua fatura. Se precisar pode utilizar os cartões presentes em nosso [roteiro de teste](http://docs.desktop.cappta.com.br/docs/portf%C3%B3lio-de-cart%C3%B5es-de-testes). Lembrando que vendas digitadas é permitido apenas para a modalidade crédito.

Vamos para a elaboração dos metodos para pagamento.

O primeiro é pagamento débito, o mais simples.

```javascript
private void OnExecutaPagamentoDebitoClick(object sender, EventArgs e)
{
	if (this.DeveIniciarMultiCartoes()) { this.IniciarMultiCartoes(); }

	double valor = (double)NumericUpDownValorPagamentoDebito.Value;

	if (this.DeveIniciarMultiCartoes()) { this.IniciarMultiCartoes(); }

	int resultado = this.cliente.PagamentoDebito(valor);
	if (resultado != 0) { this.CriarMensagemErroPainel(resultado); return; }

	this.processandoPagamento = true;
	this.IterarOperacaoTef();
}
```
<h3>Agora pagamento credito:</h3>

```javascript
private void OnExecutaPagamentoCreditoClick(object sender, EventArgs e)
{
	if (this.DeveIniciarMultiCartoes()) { this.IniciarMultiCartoes(); }

	double valor = (double)NumericUpDownValorPagamentoCredito.Value;
	IDetalhesCredito details = new DetalhesCredito
	{
		QuantidadeParcelas = (int)this.NumericUpDownQuantidadeParcelasPagamentoCredito.Value,
		TipoParcelamento = (int)this.tiposParcelamento[ComboBoxTipoParcelamentoPagamentoCredito.SelectedIndex],
		TransacaoParcelada = this.RadioButtonPagamentoCreditoComParcelas.Checked,
	};

	if (this.DeveIniciarMultiCartoes()) { this.IniciarMultiCartoes(); }

	int resultado = this.cliente.PagamentoCredito(valor, details);
	if (resultado != 0) { this.CriarMensagemErroPainel(resultado); return; }

	this.processandoPagamento = true;
	this.IterarOperacaoTef();
}
```

<h3>Crediário </h3>

```javascript
private void OnExecutaPagamentoCrediarioClick(object sender, EventArgs e)
{
	double valor = (double)NumericUpDownValorPagamentoCrediario.Value;
	IDetalhesCrediario detalhes = new DetalhesCrediario
	{
		QuantidadeParcelas = (int)NumericUpDownQuantidadeParcelasPagamentoCrediario.Value,
	};

	if (this.DeveIniciarMultiCartoes()) { this.IniciarMultiCartoes(); }

	int resultado = this.cliente.PagamentoCrediario(valor, detalhes);
	if (resultado != 0) { this.CriarMensagemErroPainel(resultado); return; }

	this.processandoPagamento = true;
	this.IterarOperacaoTef();
}
```

<h1>Etapa 5 </h1>

Tempo estimado de 01:00 hora

**Funções administrativas**

Agora que tratamos as formas de pagamento, podemos partir para as funções administrativas. 

Clientes com frequência pedem a reimpressão de um comprovante ou um cancelamento, as funções administrativas tem a função de deixar praticas e acessiveis estas funções.

<h3>Para reimpressão </h3>
Temos as seguintes formas: 

*Reimpressão por número de controle
*Reimpressão cupom lojista
*Reimpressão cupom cliente
*Reimpressão de todas as vias

```javascript
private void OnButtonExecutaReimpressaoCupomClick(object sender, EventArgs e)
{
	if (this.sessaoMultiTefEmAndamento == true)
{
this.CriarMensagemErroJanela("Não é possível reimprimir um cupom com uma sessão multitef em andamento."); return;
}

	int resultado = this.RadioButtonReimprimirUltimoCupom.Checked

		? this.cliente.ReimprimirUltimoCupom(this.tipoVia)
		: this.cliente.ReimprimirCupom(this.NumericUpDownNumeroControleReimpressaoCupom.Value.ToString("00000000000"), this.tipoVia);

	if (resultado != 0) { this.CriarMensagemErroPainel(resultado); return; }

	this.processandoPagamento = false;
	this.IterarOperacaoTef();
}

```

<h3>Para Cancelamento </h3>

Para cancelar uma transação é preciso do número de controle e da senha administrativa, esta senha é configurável no Pinpad e por padrão é: **cappta**.  O número de controle é informado na resposta da operação aprovada.

```javascript
private void OnButtonExecutaCancelamentoClick(object sender, EventArgs e)
{
		if (this.sessaoMultiTefEmAndamento == true)
	{
		this.CriarMensagemErroJanela("Não é possível cancelar um pagamento com uma sessão multitef em andamento."); return;
	}

		string senhaAdministrativa = TextBoxSenhaAdministrativaCancelamento.Text;

		if (String.IsNullOrEmpty(senhaAdministrativa)) { this.CriarMensagemErroJanela("A senha administrativa não pode ser vazia."); return; }

		string numeroControle = NumericUpDownNumeroControleCancelamento.Value.ToString("00000000000");

		int resultado = this.cliente.CancelarPagamento(senhaAdministrativa, numeroControle);
		if (resultado != 0) { this.CriarMensagemErroPainel(resultado); return; }

		this.processandoPagamento = false;
		this.IterarOperacaoTef();
}
```
<h1> Etapa 6 </h1>

Tempo estimado de 00:40 minutos

Agora que ja fizemos 80% da integração precisamos trabalhar no Multicartões.

Multicartões ou MultiTef é uma forma de passar mais de um cartão em uma transação, nossa forma de realizar esta tarefa é diferente, se cancelarmos uma venda no meio de uma transação multtef todas são canceladas.

```javascript
private void IniciarMultiCartoes()
	{
		this.quantidadeCartoes = (int)this.NumericUpDownQuantidadeDePagamentosMultiTef.Value;
		this.sessaoMultiTefEmAndamento = true;
		this.cliente.IniciarMultiCartoes(this.quantidadeCartoes);
	}

```
<h6>
Para o código completo basta clonar o repositório, qualquer dúvida entre em contato com o time de homologação e parceria Cappta.
Quando completar a integração basta acessar nossa documentação e seguir os passos do nosso [roteiro](http://docs.desktop.cappta.com.br/docs). </h6>


**Configurando e usando:**

------------------------------------------------------------

- Instale e execute o CapptaGpPlus.exe com os dados forneceidos pela equipe;

- Execute o CapptaGpPlus;

- Extraia e abra o diretório Cappta.Gp.API.CSharp.Samples-master;

- Abra o arquivo Cappta.Gp.Api.Com.Sample.exe.config (Samples\Binaries\CSharp) em um editor de texto e configure os parametros "Cnpj" e "Pdv" com os dados fornedidos para instalação do CapptaGpPlus (não alterar a Chave de Autenticação); 
**Ex.:** authenticationKey: '795180024C04479982560F61B3C2C06E', merchantCnpj: '00000000000000', checkoutNumber: 14

- Execute o Cappta.Gp.Api.Com.Sample.exe ou use o código do projeto para fazer as transações de testes.
