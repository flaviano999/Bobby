# Bobbyfrom sqlalchemy import create_engine, Column, Integer, String, Text, Float, DateTime, ForeignKey, Boolean
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker, relationship
from hashlib import sha256
from datetime import datetime
from math import radians, sin, cos, sqrt, atan2

Base = declarative_base()

# Reduced country list for brevity (full list includes all 249 ISO 3166-1 alpha-2 codes)
COUNTRIES = {
    'Afghanistan': 'AF', 'Argentina': 'AR', 'Australia': 'AU', 'Brazil': 'BR', 'Canada': 'CA',
    'China': 'CN', 'France': 'FR', 'Germany': 'DE', 'India': 'IN', 'United States': 'US'
}

class Entidade(Base):
    __tablename__ = 'entidades'
    id = Column(Integer, primary_key=True)
    nome = Column(String(255), nullable=False)
    documento = Column(String(50))
    tipo = Column(String(50))
    endereco = Column(Text)
    pais_code = Column(String(2), nullable=False)
    latitude = Column(Float)
    longitude = Column(Float)
    biometric_hash = Column(String(64))
    role = Column(String(10))
    profession = Column(String(255))

class Chamada(Base):
    __tablename__ = 'chamadas'
    id = Column(Integer, primary_key=True)
    tomador_id = Column(Integer, ForeignKey('entidades.id'), nullable=False)
    prestador_id = Column(Integer, ForeignKey('entidades.id'), nullable=False)
    duracao_horas = Column(Float, nullable=False)
    valor_hora = Column(Float, nullable=False)
    total_pagamento = Column(Float)
    data = Column(DateTime, default=datetime.utcnow)
    latitude = Column(Float)
    longitude = Column(Float)
    tomador = relationship("Entidade", foreign_keys=[tomador_id])
    prestador = relationship("Entidade", foreign_keys=[prestador_id])

class Mensagem(Base):
    __tablename__ = 'mensagens'
    id = Column(Integer, primary_key=True)
    remetente_id = Column(Integer, ForeignKey('entidades.id'), nullable=False)
    destinatario_id = Column(Integer, ForeignKey('entidades.id'), nullable=False)
    conteudo = Column(Text, nullable=False)
    data = Column(DateTime, default=datetime.utcnow)
    remetente = relationship("Entidade", foreign_keys=[remetente_id])
    destinatario = relationship("Entidade", foreign_keys=[destinatario_id])

class Notificacao(Base):
    __tablename__ = 'notificacoes'
    id = Column(Integer, primary_key=True)
    entidade_id = Column(Integer, ForeignKey('entidades.id'), nullable=False)
    tipo = Column(String(20))
    mensagem = Column(Text, nullable=False)
    data = Column(DateTime, default=datetime.utcnow)
    lida = Column(Boolean, default=False)
    entidade = relationship("Entidade")

class Pagamento(Base):
    __tablename__ = 'pagamentos'
    id = Column(Integer, primary_key=True)
    chamada_id = Column(Integer, ForeignKey('chamadas.id'), nullable=False)
    tomador_id = Column(Integer, ForeignKey('entidades.id'), nullable=False)
    valor = Column(Float, nullable=False)
    status = Column(String(20))  # 'pendente', 'concluido', 'falhou'
    data = Column(DateTime, default=datetime.utcnow)
    transacao_id = Column(String(50))  # ID da transação no gateway
    chamada = relationship("Chamada")
    tomador = relationship("Entidade")

engine = create_engine('sqlite:///cadastro.db')
Base.metadata.create_all(engine)
Session = sessionmaker(bind=engine)

class CadastroSistema:
    def __init__(self):
        self.session = Session()

    def validar_pais(self, code):
        if code.upper() in COUNTRIES.values():
            return True
        raise ValueError("Código de país inválido. Use ISO 3166-1 alpha-2.")

    def validar_coordenadas(self, latitude, longitude):
        if not (-90 <= latitude <= 90) or not (-180 <= longitude <= 180):
            raise ValueError("Latitude deve ser -90 a 90, longitude -180 a 180.")

    def obter_coordenadas(self, endereco, pais_code):
        # Simulação da API do Google Maps Geocoding
        # Em produção, use: import googlemaps; gmaps = googlemaps.Client(key='SUA_CHAVE_API')
        # result = gmaps.geocode(endereco + ", " + pais_code)
        # return result[0]['geometry']['location']['lat'], result[0]['geometry']['location']['lng']
        # Exemplo simulado:
        if "Rua Exemplo, 123, Brasil" in endereco and pais_code == "BR":
            return -23.5505, -46.6333  # São Paulo
        elif "Av. Teste, 456, Brasil" in endereco and pais_code == "BR":
            return -23.5616, -46.6558  # São Paulo (próximo)
        return None, None

    def processar_pagamento(self, tomador_id, chamada_id, valor):
        # Simulação da API do Stripe
        # Em produção, use: import stripe; stripe.api_key = 'SUA_CHAVE_API'
        # charge = stripe.Charge.create(amount=int(valor*100), currency='brl', source='tok_visa', description=f'Chamada ID {chamada_id}')
        # return charge.id, charge.status
        transacao_id = f"txn_{chamada_id}_{int(datetime.utcnow().timestamp())}"
        status = "concluido"  # Simulação: pagamento sempre concluído
        pagamento = Pagamento(
            chamada_id=chamada_id,
            tomador_id=tomador_id,
            valor=valor,
            status=status,
            transacao_id=transacao_id
        )
        self.session.add(pagamento)
        self.session.commit()
        return transacao_id, status

    def calcular_distancia(self, lat1, lon1, lat2, lon2):
        R = 6371.0
        lat1, lon1, lat2, lon2 = map(radians, [lat1, lon1, lat2, lon2])
        dlat = lat2 - lat1
        dlon = lon2 - lon1
        a = sin(dlat / 2)**2 + cos(lat1) * cos(lat2) * sin(dlon / 2)**2
        c = 2 * atan2(sqrt(a), sqrt(1 - a))
        return R * c

    def cadastrar(self, nome, documento, tipo, endereco, pais_code, biometric_data, role, profession, latitude=None, longitude=None):
        self.validar_pais(pais_code)
        if latitude is None or longitude is None:
            latitude, longitude = self.obter_coordenadas(endereco, pais_code)
        if latitude is not None and longitude is not None:
            self.validar_coordenadas(latitude, longitude)
        biometric_hash = sha256(biometric_data.encode()).hexdigest()
        entidade = Entidade(
            nome=nome, documento=documento, tipo=tipo, endereco=endereco,
            pais_code=pais_code.upper(), biometric_hash=biometric_hash,
            role=role, profession=profession, latitude=latitude, longitude=longitude
        )
        self.session.add(entidade)
        self.session.commit()
        print(f"{role.capitalize()} cadastrado: {nome} (Profissão: {profession})")

    def autenticar_biometrico(self, entidade_id, biometric_data_fornecida):
        entidade = self.session.query(Entidade).get(entidade_id)
        if not entidade:
            return False
        hash_fornecido = sha256(biometric_data_fornecida.encode()).hexdigest()
        return hash_fornecido == entidade.biometric_hash

    def registrar_chamada(self, tomador_id, prestador_id, duracao_horas, valor_hora, latitude=None, longitude=None):
        tomador = self.session.query(Entidade).get(tomador_id)
        prestador = self.session.query(Entidade).get(prestador_id)
        if not tomador or tomador.role != "tomador":
            raise ValueError("Tomador inválido.")
        if not prestador or prestador.role != "prestador":
            raise ValueError("Prestador inválido.")
        if latitude is not None and longitude is not None:
            self.validar_coordenadas(latitude, longitude)
        total_pagamento = duracao_horas * valor_hora
        chamada = Chamada(
            tomador_id=tomador_id, prestador_id=prestador_id,
            duracao_horas=duracao_horas, valor_hora=valor_hora,
            total_pagamento=total_pagamento, latitude=latitude, longitude=longitude
        )
        self.session.add(chamada)
        self.session.commit()
        # Processar pagamento
        transacao_id, status = self.processar_pagamento(tomador_id, chamada.id, total_pagamento)
        if status != "concluido":
            raise ValueError("Pagamento falhou.")
        # Criar notificações
        self.criar_notificacao(tomador_id, "chamada", f"Chamada registrada com {prestador.nome} ({duracao_horas}h, R${total_pagamento})")
        self.criar_notificacao(prestador_id, "chamada", f"Chamada registrada com {tomador.nome} ({duracao_horas}h, R${total_pagamento})")
        print(f"Chamada registrada: Tomador {tomador.nome}, Prestador {prestador.nome}, Total: {total_pagamento}")

    def criar_notificacao(self, entidade_id, tipo, mensagem):
        notificacao = Notificacao(entidade_id=entidade_id, tipo=tipo, mensagem=mensagem)
        self.session.add(notificacao)
        self.session.commit()

    def enviar_mensagem(self, remetente_id, destinatario_id, conteudo):
        remetente = self.session.query(Entidade).get(remetente_id)
        destinatario = self.session.query(Entidade).get(destinatario_id)
        if not remetente or not destinatario:
            raise ValueError("Remetente ou destinatário inválido.")
        if remetente.role == destinatario.role:
            raise ValueError("Mensagem deve ser entre tomador e prestador.")
        mensagem = Mensagem(
            remetente_id=remetente_id,
            destinatario_id=destinatario_id,
            conteudo=conteudo
        )
        self.session.add(mensagem)
        self.session.commit()
        self.criar_notificacao(destinatario_id, "mensagem", f"Nova mensagem de {remetente.nome}: {conteudo[:50]}...")
        print(f"Mensagem enviada de {remetente.nome} para {destinatario.nome}: {conteudo}")

    def listar_mensagens(self, entidade1_id, entidade2_id):
        mensagens = self.session.query(Mensagem).filter(
            ((Mensagem.remetente_id == entidade1_id) & (Mensagem.destinatario_id == entidade2_id)) |
            ((Mensagem.remetente_id == entidade2_id) & (Mensagem.destinatario_id == entidade1_id))
        ).order_by(Mensagem.data).all()
        for m in mensagens:
            remetente = self.session.query(Entidade).get(m.remetente_id)
            destinatario = self.session.query(Entidade).get(m.destinatario_id)
            print(f"{m.data}: {remetente.nome} -> {destinatario.nome}: {m.conteudo}")

    def listar_notificacoes(self, entidade_id, marcar_como_lidas=False):
        notificacoes = self.session.query(Notificacao).filter(Notificacao.entidade_id == entidade_id).order_by(Notificacao.data).all()
        for n in notificacoes:
            status = "Não lida" if not n.lida else "Lida"
            print(f"ID: {n.id}, Tipo: {n.tipo}, Mensagem: {n.mensagem}, Data: {n.data}, Status: {status}")
            if marcar_como_lidas and not n.lida:
                n.lida = True
        self.session.commit()

    def buscar_por_proximidade(self, latitude, longitude, raio_km, role=None):
        self.validar_coordenadas(latitude, longitude)
        entidades = self.session.query(Entidade).filter(Entidade.latitude.isnot(None), Entidade.longitude.isnot(None))
        if role:
            entidades = entidades.filter(Entidade.role == role)
        resultados = []
        for e in entidades:
            distancia = self.calcular_distancia(latitude, longitude, e.latitude, e.longitude)
            if distancia <= raio_km:
                resultados.append((e, distancia))
        resultados.sort(key=lambda x: x[1])
        for entidade, distancia in resultados:
            print(f"ID: {entidade.id}, Nome: {entidade.nome}, Role: {entidade.role}, Profissão: {entidade.profession}, Distância: {distancia:.2f} km")

    def listar(self):
        print("Entidades:")
        for e in self.session.query(Entidade).all():
            geo = f"Lat: {e.latitude}, Lon: {e.longitude}" if e.latitude and e.longitude else "Sem geolocalização"
            print(f"ID: {e.id}, Nome: {e.nome}, Role: {e.role}, País: {e.pais_code}, Tipo: {e.tipo}, Profissão: {e.profession}, Geo: {geo}")
        print("\nChamadas:")
        for c in self.session.query(Chamada).all():
            tomador = self.session.query(Entidade).get(c.tomador_id)
            prestador = self.session.query(Entidade).get(c.prestador_id)
            geo = f"Lat: {c.latitude}, Lon: {c.longitude}" if c.latitude and c.longitude else "Sem geolocalização"
            print(f"ID: {c.id}, Tomador: {tomador.nome}, Prestador: {prestador.nome}, Duração: {c.duracao_horas}h, Valor/h: {c.valor_hora}, Total: {c.total_pagamento}, Data: {c.data}, Geo: {geo}")
        print("\nMensagens:")
        for m in self.session.query(Mensagem).all():
            remetente = self.session.query(Entidade).get(m.remetente_id)
            destinatario = self.session.query(Entidade).get(m.destinatario_id)
            print(f"ID: {m.id}, {remetente.nome} -> {destinatario.nome}: {m.conteudo}, Data: {m.data}")
        print("\nNotificações:")
        for n in self.session.query(Notificacao).all():
            entidade = self.session.query(Entidade).get(n.entidade_id)
            status = "Não lida" if not n.lida else "Lida"
            print(f"ID: {n.id}, Para: {entidade.nome}, Tipo: {n.tipo}, Mensagem: {n.mensagem}, Data: {n.data}, Status: {status}")
        print("\nPagamentos:")
        for p in self.session.query(Pagamento).all():
            tomador = self.session.query(Entidade).get(p.tomador_id)
            print(f"ID: {p.id}, Chamada ID: {p.chamada_id}, Tomador: {tomador.nome}, Valor: {p.valor}, Status: {p.status}, Transação ID: {p.transacao_id}, Data: {p.data}")

# Exemplo de uso
if __name__ == "__main__":
    sistema = CadastroSistema()
    # Cadastro com geolocalização via Google Maps
    sistema.cadastrar(
        nome="João Silva", documento="12345678901", tipo="fisica",
        endereco="Rua Exemplo, 123, Brasil", pais_code="BR",
        biometric_data="impressao_digital_simulada_123", role="tomador",
        profession="Empresário"
    )
    sistema.cadastrar(
        nome="Maria Oliveira", documento="98765432109", tipo="fisica",
        endereco="Av. Teste, 456, Brasil", pais_code="BR",
        biometric_data="impressao_digital_simulada_456", role="prestador",
        profession="Engenheira Civil"
    )
    # Enviar mensagens
    sistema.enviar_mensagem(1, 2, "Oi, Maria, preciso de um orçamento para um projeto.")
    sistema.enviar_mensagem(2, 1, "Claro, João! Qual o escopo do projeto?")
    # Registrar chamada com pagamento
    sistema.registrar_chamada(
        tomador_id=1, prestador_id=2, duracao_horas=2.5, valor_hora=150.0,
        latitude=-23.5505, longitude=-46.6333
    )
    # Busca por proximidade
    print("\nBuscando prestadores em 10 km de São Paulo (-23.5505, -46.6333):")
    sistema.buscar_por_proximidade(latitude=-23.5505, longitude=-46.6333, raio_km=10, role="prestador")
    # Listar mensagens
    print("\nMensagens entre João e Maria:")
    sistema.listar_mensagens(1, 2)
    # Listar notificações
    print("\nNotificações para Maria (ID 2):")
    sistema.listar_notificacoes(2, marcar_como_lidas=True)
    # Autenticação e listagem geral
    autenticado = sistema.autenticar_biometrico(1, "impressao_digital_simulada_123")
    print(f"\nAutenticado: {autenticado}")
    sistema.listar()