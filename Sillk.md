---
name: motor-integracao-bibliografica
description: >
  Use esta skill quando precisar gerenciar citações e referências bibliográficas em manuscritos acadêmicos.
  Ativa quando o usuário insere chaves de citação provisórias no texto, quando pede para formatar referências segundo ABNT, APA ou Vancouver, ou quando precisa integrar metadados do Zotero ou Mendeley.
  Entrega texto com citações corrigidas automaticamente e lista de referências diagramada no formato escolhido.
---
# Skill: Motor de Integração Bibliográfica (Zotero/Mendeley)

## Propósito
Automatizar completamente o trato de citações e referências bibliográficas em manuscritos acadêmicos, desde a ingestão de metadados até a diagramação da lista final. A skill integra-se com Zotero e Mendeley via APIs, processa arquivos BibTeX/RIS, substitui chaves de citação provisórias no texto e aplica normas acadêmicas (ABNT, APA, Vancouver) com precisão absoluta.

## O que esta skill NÃO faz
- Não busca artigos em bases de dados (Scopus, Web of Science) — apenas processa metadados já obtidos
- Não traduz automaticamente referências entre idiomas
- Não gera citações automáticas a partir de conteúdo textual
- Não substitui gerenciadores de referência — complementa a formatação final

## Fontes
- Referência: `Motor de Integração Bibliográfica.pdf` (skill externa EscreveAI)
- Internet: Documentação da API do Zotero (https://www.zotero.org/support/dev/web_api/v3/start)
- Internet: Documentação da API do Mendeley (https://dev.mendeley.com/)
- Internet: Especificações BibTeX (https://www.bibtex.org/Format/)
- Internet: Formatos RIS (https://en.wikipedia.org/wiki/RIS_(file_format))

## Pré-requisitos
- Python 3.10+
- `bibtexparser` (pip install bibtexparser)
- `requests` (pip install requests)
- `json` (biblioteca padrão)
- `re` (biblioteca padrão)
- `datetime` (biblioteca padrão)
- Chaves de API do Zotero e/ou Mendeley (opcional, para modo online)

## Arquitetura de Dados

### Estrutura Interna de Metadados
```python
from dataclasses import dataclass, field
from typing import List, Optional
from enum import Enum

class NormaAcademica(Enum):
    ABNT = "abnt"
    APA = "apa"
    VANCOUVER = "vancouver"

class TipoReferencia(Enum):
    ARTIGO = "artigo"
    LIVRO = "livro"
    CAPITULO_LIVRO = "capitulo_livro"
    TESE = "tese"
    TRABALHO_CONFERENCIA = "trabalho_conferencia"
    RELATORIO = "relatorio"
    LEI = "lei"
    PAGINA_WEB = "pagina_web"

@dataclass
class MetadadosBibliograficos:
    """Estrutura unificada de metadados bibliográficos."""
    id_unico: str
    tipo: TipoReferencia
    autores: List[str] = field(default_factory=list)
    ano: Optional[int] = None
    titulo: Optional[str] = None
    titulo_alternativo: Optional[str] = None  # Para traduções
    periódico: Optional[str] = None
    volume: Optional[str] = None
    numero: Optional[str] = None
    paginas: Optional[str] = None
    doi: Optional[str] = None
    url: Optional[str] = None
    editora: Optional[str] = None
    local: Optional[str] = None
    edicao: Optional[str] = None
    capitulos: Optional[str] = None  # Para capítulos de livro
    orientador: Optional[str] = None  # Para teses
    instituicao: Optional[str] = None  # Para teses
    nome_conferencia: Optional[str] = None
    local_conferencia: Optional[str] = None
    data_publicacao: Optional[str] = None
    idioma: Optional[str] = None
    abstract: Optional[str] = None
    palavras_chave: List[str] = field(default_factory=list)
    notas: Optional[str] = None
    acessado_em: Optional[str] = None  # Para páginas web
    fonte: Optional[str] = None  # Zotero, Mendeley, BibTeX, RIS
```

## Fluxo de Execução

### Passo 1: Ingestão de Metadados
```python
import bibtexparser
import json
import requests
from pathlib import Path

class IngestorBibliografico:
    """Motor de ingestão de metadados bibliográficos."""
    
    def __init__(self, zotero_api_key: str = None, mendeley_token: str = None):
        self.zotero_api_key = zotero_api_key
        self.mendeley_token = mendeley_token
        self.cache = {}  # Cache local de metadados
    
    def ingerir(self, fonte: str, **kwargs) -> List[MetadadosBibliograficos]:
        """
        Ingere metadados de múltiplas fontes.
        
        Args:
            fonte: 'bibtex', 'ris', 'zotero', 'mendeley', 'arquivo'
            **kwargs: Parâmetros específicos da fonte
        """
        if fonte == 'bibtex':
            return self._ingerir_bibtex(kwargs['caminho'])
        elif fonte == 'ris':
            return self._ingerir_ris(kwargs['caminho'])
        elif fonte == 'zotero':
            return self._ingerir_zotero(kwargs['user_id'], kwargs.get('collection_key'))
        elif fonte == 'mendeley':
            return self._ingerir_mendeley(kwargs.get('folder_id'))
        elif fonte == 'arquivo':
            return self._ingerir_arquivo_generico(kwargs['caminho'])
        else:
            raise ValueError(f"Fonte não suportada: {fonte}")
    
    def _ingerir_bibtex(self, caminho: str) -> List[MetadadosBibliograficos]:
        """Ingere arquivo BibTeX (.bib)."""
        try:
            with open(caminho, 'r', encoding='utf-8') as f:
                conteudo = f.read()
            
            # Parse do BibTeX
            banco = bibtexparser.loads(conteudo)
            
            metadados_lista = []
            for entrada in banco.entries:
                metadados = self._converter_bibtex_para_metadados(entrada)
                metadados_lista.append(metadados)
                self.cache[metadados.id_unico] = metadados
            
            return metadados_lista
            
        except Exception as e:
            raise RuntimeError(f"Erro ao ler BibTeX: {e}")
    
    def _converter_bibtex_para_metadados(self, entrada: dict) -> MetadadosBibliograficos:
        """Converte entrada BibTeX para estrutura unificada."""
        # Mapeamento de tipos BibTeX
        mapeamento_tipos = {
            'article': TipoReferencia.ARTIGO,
            'book': TipoReferencia.LIVRO,
            'inbook': TipoReferencia.CAPITULO_LIVRO,
            'incollection': TipoReferencia.CAPITULO_LIVRO,
            'phdthesis': TipoReferencia.TESE,
            'mastersthesis': TipoReferencia.TESE,
            'inproceedings': TipoReferencia.TRABALHO_CONFERENCIA,
            'techreport': TipoReferencia.RELATORIO,
            'misc': TipoReferencia.PAGINA_WEB
        }
        
        tipo = mapeamento_tipos.get(entrada.get('type', 'misc'), TipoReferencia.PAGINA_WEB)
        
        # Processar autores (formato BibTeX: "Sobrenome, Nome e Sobrenome, Nome")
        autores_raw = entrada.get('author', '')
        autores = self._processar_autores_bibtex(autores_raw)
        
        # Extrair ano
        ano = None
        if 'year' in entrada:
            try:
                ano = int(entrada['year'])
            except ValueError:
                pass
        
        return MetadadosBibliograficos(
            id_unico=entrada.get('id', entrada.get('key', '')),
            tipo=tipo,
            autores=autores,
            ano=ano,
            titulo=entrada.get('title', ''),
            titulo_alternativo=entrada.get('subtitle', ''),
            periódico=entrada.get('journal', ''),
            volume=entrada.get('volume', ''),
            numero=entrada.get('number', ''),
            paginas=entrada.get('pages', ''),
            doi=entrada.get('doi', ''),
            url=entrada.get('url', ''),
            editora=entrada.get('publisher', ''),
            local=entrada.get('address', ''),
            edicao=entrada.get('edition', ''),
            orientador=entrada.get('school', '') if tipo == TipoReferencia.TESE else None,
            instituicao=entrada.get('institution', '') if tipo == TipoReferencia.RELATORIO else None,
            idioma=entrada.get('language', ''),
            abstract=entrada.get('abstract', ''),
            notas=entrada.get('note', ''),
            fonte='bibtex'
        )
    
    def _processar_autores_bibtex(self, autores_raw: str) -> List[str]:
        """Processa string de autores no formato BibTeX."""
        if not autores_raw:
            return []
        
        # Separar por ' e ' (padrão BibTeX)
        autores = autores_raw.split(' and ')
        
        # Limpar espaços
        autores = [autor.strip() for autor in autores]
        
        return autores
    
    def _ingerir_ris(self, caminho: str) -> List[MetadadosBibliograficos]:
        """Ingere arquivo RIS."""
        try:
            with open(caminho, 'r', encoding='utf-8') as f:
                conteudo = f.read()
            
            # Parse do RIS
            registros = self._parse_ris(conteudo)
            
            metadados_lista = []
            for registro in registros:
                metadados = self._converter_ris_para_metadados(registro)
                metadados_lista.append(metadados)
                self.cache[metadados.id_unico] = metadados
            
            return metadados_lista
            
        except Exception as e:
            raise RuntimeError(f"Erro ao ler RIS: {e}")
    
    def _parse_ris(self, conteudo: str) -> List[dict]:
        """Parse de formato RIS."""
        registros = []
        registro_atual = {}
        
        for linha in conteudo.split('\n'):
            linha = linha.strip()
            
            if linha.startswith('TY  - '):
                if registro_atual:
                    registros.append(registro_atual)
                registro_atual = {'tipo': linha[6:]}
            elif linha.startswith('ER  - '):
                if registro_atual:
                    registros.append(registro_atual)
                registro_atual = {}
            elif '  - ' in linha:
                chave, valor = linha.split('  - ', 1)
                if chave in registro_atual:
                    # Múltiplos valores da mesma chave
                    if isinstance(registro_atual[chave], list):
                        registro_atual[chave].append(valor)
                    else:
                        registro_atual[chave] = [registro_atual[chave], valor]
                else:
                    registro_atual[chave] = valor
        
        return registros
    
    def _converter_ris_para_metadados(self, registro: dict) -> MetadadosBibliograficos:
        """Converte registro RIS para estrutura unificada."""
        # Mapeamento de tipos RIS
        mapeamento_tipos = {
            'JOUR': TipoReferencia.ARTIGO,
            'BOOK': TipoReferencia.LIVRO,
            'CHAP': TipoReferencia.CAPITULO_LIVRO,
            'THES': TipoReferencia.TESE,
            'CONF': TipoReferencia.TRABALHO_CONFERENCIA,
            'RPRT': TipoReferencia.RELATORIO,
            'ELEC': TipoReferencia.PAGINA_WEB,
            'WEB': TipoReferencia.PAGINA_WEB
        }
        
        tipo = mapeamento_tipos.get(registro.get('tipo', ''), TipoReferencia.PAGINA_WEB)
        
        # Processar autores
        autores_raw = registro.get('AU', [])
        if isinstance(autores_raw, str):
            autores_raw = [autores_raw]
        
        # Extrair ano
        ano = None
        if 'PY' in registro:
            try:
                ano = int(registro['PY'])
            except ValueError:
                pass
        
        return MetadadosBibliograficos(
            id_unico=registro.get('ID', str(hash(json.dumps(registro, sort_keys=True)))),
            tipo=tipo,
            autores=autores_raw,
            ano=ano,
            titulo=registro.get('TI', registro.get('T1', '')),
            titulo_alternativo=registro.get('T2', ''),
            periódico=registro.get('JO', registro.get('JA', '')),
            volume=registro.get('VL', ''),
            numero=registro.get('IS', ''),
            paginas=registro.get('SP', ''),
            doi=registro.get('DO', ''),
            url=registro.get('UR', ''),
            editora=registro.get('PB', ''),
            local=registro.get('CY', ''),
            idioma=registro.get('LA', ''),
            abstract=registro.get('AB', ''),
            notas=registro.get('N1', ''),
            fonte='ris'
        )
```

### Passo 2: Parser de Citações no Texto
```python
class ParserCitacoes:
    """Motor de varredura e substituição dinâmica de citações."""
    
    def __init__(self, metadados: List[MetadadosBibliograficos]):
        self.metadados = {m.id_unico: m for m in metadados}
        self.indice_autores = self._construir_indice_autores()
        self.indice_chaves = self._construir_indice_chaves()
    
    def _construir_indice_autores(self) -> dict:
        """Constrói índice para busca por autor."""
        indice = {}
        for m in self.metadados.values():
            for autor in m.autores:
                sobrenome = autor.split(',')[0].strip().upper()
                if sobrenome not in indice:
                    indice[sobrenome] = []
                indice[sobrenome].append(m)
        return indice
    
    def _construir_indice_chaves(self) -> dict:
        """Constrói índice para busca por chave."""
        indice = {}
        for m in self.metadados.values():
            # Chave primária
            indice[m.id_unico.lower()] = m
            
            # Chaves alternativas
            if m.autores and m.ano:
                sobrenome = m.autores[0].split(',')[0].strip().lower()
                chave_alternativa = f"{sobrenome}{m.ano}"
                indice[chave_alternativa] = m
                
                # Para desambiguação (a, b, c)
                for i, autor in enumerate(m.autores):
                    if i > 0:
                        sobrenome = autor.split(',')[0].strip().lower()
                        chave_alternativa = f"{sobrenome}{m.ano}"
                        indice[chave_alternativa] = m
        
        return indice
    
    def processar_texto(self, texto: str, norma: NormaAcademica) -> str:
        """
        Processa texto completo, substituindo chaves de citação provisórias.
        
        Padrões suportados:
        - [chave] - citação entre parênteses
        - [chave:página] - citação direta com página
        - {chave} - alternativa para [chave]
        """
        # Padrões regex para identificar citações
        padroes = [
            (r'\[([^\]]+):(\d+)\]', self._substituir_citacao_direta),
            (r'\[([^\]]+)\]', self._substituir_citacao_indireta),
            (r'\{([^}]+):(\d+)\}', self._substituir_citacao_direta),
            (r'\{([^}]+)\}', self._substituir_citacao_indireta)
        ]
        
        resultado = texto
        for padrao, funcao in padroes:
            resultado = re.sub(padrao, funcao, resultado)
        
        return resultado
    
    def _substituir_citacao_indireta(self, match) -> str:
        """Substitui citação indireta [chave] por formato da norma."""
        chave = match.group(1)
        metadados = self._buscar_chave(chave)
        
        if not metadados:
            return match.group(0)  # Mantém original se não encontrar
        
        # Formatação conforme norma
        return self._formatar_citacao_indireta(metadados)
    
    def _substituir_citacao_direta(self, match) -> str:
        """Substitui citação direta [chave:página] por formato da norma."""
        chave = match.group(1)
        pagina = match.group(2)
        metadados = self._buscar_chave(chave)
        
        if not metadados:
            return match.group(0)
        
        return self._formatar_citacao_direta(metadados, pagina)
    
    def _buscar_chave(self, chave: str) -> Optional[MetadadosBibliograficos]:
        """Busca metadados por chave."""
        chave_lower = chave.lower()
        
        # Busca exata
        if chave_lower in self.indice_chaves:
            return self.indice_chaves[chave_lower]
        
        # Busca por autor + ano
        match = re.match(r'^([a-zA-Z]+)(\d{4})$', chave)
        if match:
            sobrenome = match.group(1).lower()
            ano = match.group(2)
            chave_busca = f"{sobrenome}{ano}"
            if chave_busca in self.indice_chaves:
                return self.indice_chaves[chave_busca]
        
        return None
    
    def _formatar_citacao_indireta(self, metadados: MetadadosBibliograficos) -> str:
        """Formata citação indireta conforme norma."""
        autores = metadados.autores
        ano = metadados.ano
        
        if not autores:
            return f"({ano})"
        
        sobrenomes = [a.split(',')[0].strip() for a in autores]
        
        if len(sobrenomes) == 1:
            return f"({sobrenomes[0]}, {ano})"
        elif len(sobrenomes) == 2:
            return f"({sobrenomes[0]} e {sobrenomes[1]}, {ano})"
        elif len(sobrenomes) <= 3:
            return f"({', '.join(sobrenomes[:-1])} e {sobrenomes[-1]}, {ano})"
        else:
            # Mais de 3 autores: et al. (APA) ou outros (ABNT/Vancouver)
            return f"({sobrenomes[0]} et al., {ano})"
    
    def _formatar_citacao_direta(self, metadados: MetadadosBibliograficos, pagina: str) -> str:
        """Formata citação direta conforme norma."""
        autores = metadados.autores
        ano = metadados.ano
        
        if not autores:
            return f"({ano}, p. {pagina})"
        
        sobrenomes = [a.split(',')[0].strip() for a in autores]
        
        if len(sobrenomes) == 1:
            return f"({sobrenomes[0]}, {ano}, p. {pagina})"
        elif len(sobrenomes) == 2:
            return f"({sobrenomes[0]} e {sobrenomes[1]}, {ano}, p. {pagina})"
        else:
            return f"({sobrenomes[0]} et al., {ano}, p. {pagina})"
```

### Passo 3: Motor de Enquadramento Normativo
```python
class MotorNormativo:
    """Motor de formatação conforme normas acadêmicas."""
    
    def __init__(self, norma: NormaAcademica):
        self.norma = norma
        self.regras = self._carregar_regras()
    
    def _carregar_regras(self) -> dict:
        """Carrega regras de formatação para a norma."""
        if self.norma == NormaAcademica.ABNT:
            return self._regras_abnt()
        elif self.norma == NormaAcademica.APA:
            return self._regras_apa()
        elif self.norma == NormaAcademica.VANCOUVER:
            return self._regras_vancouver()
        else:
            raise ValueError(f"Norma não suportada: {self.norma}")
    
    def _regras_abnt(self) -> dict:
        """Regras da ABNT NBR 6023:2018."""
        return {
            'autores_max_et_al': 2,  # Até 3 autores: listar todos
            'separador_autores': '; ',
            'ultimo_separador': ' e ',
            'maiusculas_dentro_parenteses': True,
            'formato_autor': '{sobrenome}, {inicial}.',
            'formato_citacao_corpo': '({sobrenome}, {ano})',
            'formato_citacao_notas': '{sobrenome}, {ano}. {titulo}. {local}: {editora}, {ano}.',
            'formato_lista': 'Ordem alfabética',
            'recuo_suspensivo': 'Sim (1,5 cm)',
            'separador_paginas': '-'
        }
    
    def _regras_apa(self) -> dict:
        """Regras da APA 7ª edição."""
        return {
            'autores_max_et_al': 5,  # Até 5 autores: listar todos; 6+: et al. na primeira
            'separador_autores': ', ',
            'ultimo_separador': ', & ',
            'maiusculas_dentro_parenteses': False,
            'formato_autor': '{sobrenome}, {inicial}.',
            'formato_citacao_corpo': '({sobrenome}, {ano})',
            'formato_citacao_notas': '{sobrenome}, {inicial}. ({ano}). {titulo}. {editora}.',
            'formato_lista': 'Ordem alfabética',
            'recuo_suspensivo': 'Sim (0,5 polegadas)',
            'separador_paginas': '-'
        }
    
    def _regras_vancouver(self) -> dict:
        """Regras do Vancouver (ICMJE)."""
        return {
            'autores_max_et_al': 6,  # Até 6: listar todos; 7+: et al. após 6º
            'separador_autores': ', ',
            'ultimo_separador': ', ',
            'maiusculas_dentro_parenteses': False,
            'formato_autor': '{sobrenome} {inicial}',
            'formato_citacao_corpo': '{numero}',
            'formato_citacao_notas': '{sobrenome} {inicial}. {titulo}. {local}: {editora}; {ano}.',
            'formato_lista': 'Ordem numérica (primeira citação)',
            'recuo_suspensivo': 'Não',
            'separador_paginas': '-'
        }
    
    def formatar_autor(self, nome_completo: str, posicao: int = 0) -> str:
        """Formata nome do autor conforme norma."""
        partes = nome_completo.split(',')
        sobrenome = partes[0].strip()
        nome = partes[1].strip() if len(partes) > 1 else ''
        
        if self.norma == NormaAcademica.VANCOUVER:
            # Vancouver: Sobrenome + Inicial (sem ponto)
            iniciais = ''.join([p[0].upper() for p in nome.split() if p])
            return f"{sobrenome} {iniciais}"
        else:
            # ABNT e APA: Sobrenome, Inicial.
            iniciais = '. '.join([p[0].upper() for p in nome.split() if p]) + '.'
            return f"{sobrenome}, {iniciais}"
    
    def formatar_lista_autores(self, autores: List[str]) -> str:
        """Formata lista de autores conforme regras da norma."""
        if not autores:
            return ""
        
        regras = self.regras
        max_et_al = regras['autores_max_et_al']
        
        if len(autores) <= max_et_al:
            # Listar todos
            autores_formatados = [self.formatar_autor(a, i) for i, a in enumerate(autores)]
            
            if len(autores) == 1:
                return autores_formatados[0]
            elif len(autores) == 2:
                return f"{autores_formatados[0]}{regras['ultimo_separador']}{autores_formatados[1]}"
            else:
                return f"{', '.join(autores_formatados[:-1])}{regras['ultimo_separador']}{autores_formatados[-1]}"
        else:
            # Usar et al.
            autor_primeiro = self.formatar_autor(autores[0])
            
            if self.norma == NormaAcademica.APA and len(autores) <= 20:
                # APA 7: listar até 20 autores
                autores_formatados = [self.formatar_autor(a) for a in autores]
                return f"{', '.join(autores_formatados[:-1])}, & {autores_formatados[-1]}"
            else:
                return f"{autor_primeiro}, et al."
    
    def desambiguar_autores(self, autores: List[MetadadosBibliograficos]) -> List[MetadadosBibliograficos]:
        """Desambigua autores com mesmo sobrenome."""
        # Agrupar por sobrenome
        por_sobrenome = {}
        for m in autores:
            if m.autores:
                sobrenome = m.autores[0].split(',')[0].strip().upper()
                if sobrenome not in por_sobrenome:
                    por_sobrenome[sobrenome] = []
                por_sobrenome[sobrenome].append(m)
        
        # Adicionar sufixos para mesmo autor/mesmo ano
        for sobrenome, lista in por_sobrenome.items():
            if len(lista) > 1:
                # Ordenar por ano
                lista_ordenada = sorted(lista, key=lambda x: x.ano or 0)
                
                # Adicionar sufixos a, b, c
                for i, m in enumerate(lista_ordenada):
                    if m.ano:
                        # Modificar ID para incluir sufixo
                        sufixo = chr(ord('a') + i)
                        m.id_unico = f"{m.id_unico}{sufixo}"
        
        return autores
```

### Passo 4: Compilação da Lista de Referências
```python
class CompiladorReferencias:
    """Motor de compilação e diagramação da lista final."""
    
    def __init__(self, norma: NormaAcademica):
        self.norma = norma
        self.motor_normativo = MotorNormativo(norma)
    
    def compilar_lista(self, metadados: List[MetadadosBibliograficos], 
                       ordenacao: str = 'alfabetica') -> str:
        """Compila lista final de referências."""
        
        # Desambiguar autores
        metadados = self.motor_normativo.desambiguar_autores(metadados)
        
        # Ordenar
        if ordenacao == 'alfabetica':
            metadados_ordenados = sorted(metadados, 
                                        key=lambda x: (x.autores[0].split(',')[0].upper() if x.autores else '', 
                                                       x.ano or 0))
        elif ordenacao == 'numerica':
            metadados_ordenados = metadados  # Mantém ordem de citação
        else:
            raise ValueError(f"Ordenação não suportada: {ordenacao}")
        
        # Formatar cada referência
        linhas = []
        for m in metadados_ordenados:
            referencia_formatada = self._formatar_referencia(m)
            linhas.append(referencia_formatada)
        
        # Aplicar formatação visual
        resultado = self._aplicar_formatacao_visual(linhas)
        
        return resultado
    
    def _formatar_referencia(self, m: MetadadosBibliograficos) -> str:
        """Formata uma única referência conforme norma."""
        
        if self.norma == NormaAcademica.ABNT:
            return self._formatar_abnt(m)
        elif self.norma == NormaAcademica.APA:
            return self._formatar_apa(m)
        elif self.norma == NormaAcademica.VANCOUVER:
            return self._formatar_vancouver(m)
        else:
            raise ValueError(f"Norma não suportada: {self.norma}")
    
    def _formatar_abnt(self, m: MetadadosBibliograficos) -> str:
        """Formata referência conforme ABNT."""
        autores = self.motor_normativo.formatar_lista_autores(m.autores)
        
        if m.tipo == TipoReferencia.ARTIGO:
            return (f"{autores}. {m.titulo}. "
                   f"{m.periódico}, {m.local}, v. {m.volume}, n. {m.numero}, "
                   f"p. {m.paginas}, {m.ano}.")
        elif m.tipo == TipoReferencia.LIVRO:
            return (f"{autores}. {m.titulo}. "
                   f"{m.edicao}. {m.local}: {m.editora}, {m.ano}.")
        elif m.tipo == TipoReferencia.CAPITULO_LIVRO:
            return (f"{autores}. {m.titulo}. "
                   f"In: {m.periódico}. {m.local}: {m.editora}, {m.ano}. p. {m.paginas}.")
        elif m.tipo == TipoReferencia.TESE:
            return (f"{autores}. {m.titulo}. "
                   f"{m.local}: {m.instituicao}, {m.ano}.")
        elif m.tipo == TipoReferencia.TRABALHO_CONFERENCIA:
            return (f"{autores}. {m.titulo}. "
                   f"{m.nome_conferencia}, {m.local_conferencia}, {m.ano}.")
        elif m.tipo == TipoReferencia.PAGINA_WEB:
            return (f"{autores}. {m.titulo}. "
                   f"{m.url}. Acesso em: {m.acessado_em}.")
        else:
            return f"{autores}. {m.titulo}. {m.ano}."
    
    def _formatar_apa(self, m: MetadadosBibliograficos) -> str:
        """Formata referência conforme APA."""
        autores = self.motor_normativo.formatar_lista_autores(m.autores)
        
        if m.tipo == TipoReferencia.ARTIGO:
            return (f"{autores} ({m.ano}). {m.titulo}. "
                   f"{m.periódico}, {m.volume}({m.numero}), {m.paginas}. "
                   f"https://doi.org/{m.doi}" if m.doi else 
                   f"{autores} ({m.ano}). {m.titulo}. "
                   f"{m.periódico}, {m.volume}({m.numero}), {m.paginas}.")
        elif m.tipo == TipoReferencia.LIVRO:
            return (f"{autores} ({m.ano}). {m.titulo}. "
                   f"{m.editora}.")
        elif m.tipo == TipoReferencia.CAPITULO_LIVRO:
            return (f"{autores} ({m.ano}). {m.titulo}. "
                   f"In {m.periódico} (pp. {m.paginas}). {m.editora}.")
        elif m.tipo == TipoReferencia.TESE:
            return (f"{autores} ({m.ano}). {m.titulo} [Tese de doutorado, {m.instituicao}]. "
                   f"{m.local}.")
        elif m.tipo == TipoReferencia.PAGINA_WEB:
            return (f"{autores} ({m.ano}). {m.titulo}. "
                   f"{m.url}")
        else:
            return f"{autores} ({m.ano}). {m.titulo}."
    
    def _formatar_vancouver(self, m: MetadadosBibliograficos) -> str:
        """Formata referência conforme Vancouver."""
        autores = self.motor_normativo.formatar_lista_autores(m.autores)
        
        if m.tipo == TipoReferencia.ARTIGO:
            return (f"{autores}. {m.titulo}. "
                   f"{m.periódico}. {m.ano};{m.volume}({m.numero}):{m.paginas}.")
        elif m.tipo == TipoReferencia.LIVRO:
            return (f"{autores}. {m.titulo}. "
                   f"{m.local}: {m.editora}; {m.ano}.")
        elif m.tipo == TipoReferencia.TESE:
            return (f"{autores}. {m.titulo} "
                   f"[Tese]. {m.local}: {m.instituicao}; {m.ano}.")
        elif m.tipo == TipoReferencia.TRABALHO_CONFERENCIA:
            return (f"{autores}. {m.titulo}. "
                   f"{m.nome_conferencia}. {m.local}; {m.ano}.")
        elif m.tipo == TipoReferencia.PAGINA_WEB:
            return (f"{autores}. {m.titulo}. "
                   f"[Internet]. {m.url}; {m.ano} [citado {m.acessado_em}].")
        else:
            return f"{autores}. {m.titulo}. {m.ano}."
    
    def _aplicar_formatacao_visual(self, linhas: List[str]) -> str:
        """Aplica formatação visual (recuo, alinhamento)."""
        resultado = []
        
        for i, linha in enumerate(linhas):
            # ABNT: recuo de 1,5 cm na segunda linha em diante
            if self.norma == NormaAcademica.ABNT and i > 0:
                # Simular recuo com espaços (1,5 cm ≈ 4 espaços)
                linha = "    " + linha
            
            resultado.append(linha)
        
        return "\n\n".join(resultado)
```

### Passo 5: Pipeline Completo
```python
class PipelineBibliografico:
    """Pipeline completo de integração bibliográfica."""
    
    def __init__(self, norma: NormaAcademica = NormaAcademica.APA):
        self.norma = norma
        self.ingestor = IngestorBibliografico()
        self.cache = {}
    
    def executar(self, 
                 caminho_texto: str,
                 fontes_bibliograficas: List[dict],
                 caminho_saida: str = None) -> str:
        """
        Executa pipeline completo.
        
        Args:
            caminho_texto: Caminho para arquivo de texto com chaves provisórias
            fontes_bibliograficas: Lista de fontes [{tipo: 'bibtex', caminho: '...'}]
            caminho_saida: Caminho para salvar resultado (opcional)
        
        Returns:
            Texto com citações corrigidas
        """
        # 1. Ingerir metadados de todas as fontes
        todos_metadados = []
        for fonte in fontes_bibliograficas:
            metadados = self.ingestor.ingerir(**fonte)
            todos_metadados.extend(metadados)
        
        # 2. Inicializar parser de citações
        parser = ParserCitacoes(todos_metadados)
        
        # 3. Ler texto de entrada
        with open(caminho_texto, 'r', encoding='utf-8') as f:
            texto_original = f.read()
        
        # 4. Processar citações
        texto_processado = parser.processar_texto(texto_original, self.norma)
        
        # 5. Compilar lista de referências
        compilador = CompiladorReferencias(self.norma)
        lista_referencias = compilador.compilar_lista(todos_metadados)
        
        # 6. Montar documento final
        documento_final = self._montar_documento_final(texto_processado, lista_referencias)
        
        # 7. Salvar se solicitado
        if caminho_saida:
            with open(caminho_saida, 'w', encoding='utf-8') as f:
                f.write(documento_final)
        
        return documento_final
    
    def _montar_documento_final(self, texto: str, referencias: str) -> str:
        """Monta documento final com corpo e referências."""
        return f"""{texto}

---

## Referências

{referencias}
"""
```

## Tratamento de Erros
- **Arquivo BibTeX/RIS não encontrado**: Verificar caminho e permissões
- **Formato inválido**: Usar `_parse_texto_generico()` com regex flexível
- **Chave de citação não encontrada**: Manter chave original e informar no log
- **API Zotero/Mendeley falha**: Usar cache local ou modo offline
- **Norma não suportada**: Usar APA como padrão
- **Autor com formato inesperado**: Tentar parsing flexível, senão usar formato literal

## Regras
### O que SEMPRE fazer
- Preservar chaves de citação originais caso não encontre correspondência
- Informar ao usuário quais citações não puderam ser resolvidas
- Manter cache de metadados para performance
- Suportar múltiplas fontes bibliográficas simultaneamente
- Gerar lista de referências completa mesmo com citações não resolvidas

### O que NUNCA fazer
- Alterar conteúdo acadêmico do texto
- Inventar metadados que não existem nas fontes
- Omitir referências citadas no texto
- Alterar numeração de páginas em citações diretas
- Misturar normas no mesmo documento

### Quando algo falhar
1. **Chave não encontrada**: Manter `[chave_original]` e informar no log
2. **API indisponível**: Usar modo offline com cache existente
3. **Formato de arquivo inválido**: Tentar parsing genérico ou pedir arquivo correto
4. **Norma não definida**: Usar APA como padrão internacional

## Exemplos de Uso

### Exemplo 1: Processar texto com BibTeX
```python
pipeline = PipelineBibliografico(norma=NormaAcademica.APA)
resultado = pipeline.executar(
    caminho_texto='manuscrito.md',
    fontes_bibliograficas=[
        {'fonte': 'bibtex', 'caminho': 'referencias.bib'}
    ],
    caminho_saida='manuscrito_final.md'
)
```

### Exemplo 2: Integração com Zotero
```python
pipeline = PipelineBibliografico(norma=NormaAcademica.ABNT)
pipeline.ingestor = IngestorBibliografico(zotero_api_key='SUA_CHAVE')
resultado = pipeline.executar(
    caminho_texto='artigo.txt',
    fontes_bibliograficas=[
        {'fonte': 'zotero', 'user_id': '1234567'}
    ]
)
```

### Exemplo 3: Múltiplas fontes
```python
pipeline = PipelineBibliografico(norma=NormaAcademica.VANCOUVER)
resultado = pipeline.executar(
    caminho_texto='tese.md',
    fontes_bibliograficas=[
        {'fonte': 'bibtex', 'caminho': 'referencias.bib'},
        {'fonte': 'ris', 'caminho': 'mendeley_export.ris'},
        {'fonte': 'arquivo', 'caminho': 'outras_fontes.json'}
    ]
)
```
