d<template>
  <div class="pag">
    <Menu />
    <Perfil />
    <Mensagem
      :msg="msg"
      v-if="msg.visivel" 
      v-on:ok="msg.visivel = false"
      v-on:sairAula="sairSala(), msg.visivel = false"/>
    <div class="centro">
    <div id="botao"> </div>
      <div v-if="sala.id">
        <div class="cima">
          <div class="infoSala">
            <p class="nomeSala"><b>Sala:</b> {{sala.nome}} </p>
            <p><b>Código da sala:</b> {{sala.codigo}}</p>
          </div>
          <div class="infoSala">
            <p class="nomeProf"> <b>Professor:</b> {{sala.professor}}</p>
            <img src="../../images/sair.png" v-on:click="msgSairDaSala()" alt="Sair da Sala" class="icone" />
          </div>
        </div>
        <div class="lista" v-if="postagens[0] != null">
          <div v-for="(post, i) in postagens" v-bind:key="i">
            <Comunicado v-if="post.tipo == 'comunicado'" :comunicado="post" :professor="prof"/> 
            <div v-on:click="redirecionar(post)">
              <div v-if="post.tipo == 'atividade'" :class="[{'conc' : post.concluido == true}, {'n-conc' : post.concluido == false}]">
                {{post.concluido?"Concluída":"Pendente"}} {{post.concluido?"• Nota: "+post.nota+"%":""}}
               </div>  
              <Atividade v-if="post.tipo == 'atividade'" :atividade="post" :professor="prof"/>
            </div>
          </div>
        </div>
        <div class="sala_vazia" v-else> 
            <p class="texto"> Sala vazia</p>
            <img alt="" class="transparente" src="../../images/salavazia.png">
        </div>
      </div>
      <div v-else class="inicio">
        <div class="quadrado">
          <p class="p-else">Ingresse em uma sala!</p>
          <img src="../../images/turma.png">
          <div class="cod-sala">
            <input
              type="text"
              id="email"
              placeholder="Código da sala"
              v-model="sala.codigo"
              maxlength=""
              required
            >
            <button class="botao-entrar" v-on:click="entrarSala(sala.codigo)">Entrar</button>
          </div>
        </div>
        <div class="quadrado">
          <p class="p-else">Se torne professor!</p>
          <img src="../../images/tornarprof.png">
          <button class="botao" v-on:click="$router.push('compras')">Tornar-se</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script src="https://ajax.googleapis.com/ajax/libs/jquery/3.4.1/jquery.min.js"></script>
<script>
import Perfil from "../shared/perfil/Perfil.vue";
import Menu from "../shared/menu/Menu.vue";
import Comunicado from "../shared/comunicado/Comunicado.vue";
import Atividade from "../shared/atividade/Atividade.vue";
import Mensagem from "../shared/mensagem/Mensagem.vue";

export default {
  components: {
    Perfil,
    Menu,
    Comunicado,
    Atividade,
    Mensagem
  },
  data(){
    return {
      sala: {},
      usuario: {},
      postagens: [],
      msg: {
        visivel: false,
        titulo: "",
        mensagem: "",
        botoes: []
      },
      prof:{
        nome:'',
        foto:''
      },
   };
  },
  methods: {
    sairSala(){
      this.$http.get("https://localhost:5001/api/usuarios/" + this.$session.get("id"))
        .then(response => {
          this.usuario = response.body;
          this.usuario.idSala = 1;
          this.$session.set("idSala", 1);
          this.$session.set("usuario", this.usuario);
          
          this.$http.put("https://localhost:5001/api/usuarios/" + this.usuario.id, this.usuario)
            .then(response => {
              this.sala = {};
              this.msg = {
                visivel: true,
                titulo: "Sucesso",
                mensagem: "Você saiu de sua sala!",
                botoes: [{
                  mensagem: "OK",
                  evento: "ok"
                }]
              };
            });
        });
    },
    msgSairDaSala(){
      this.msg = {
        visivel: true,
        titulo: "Sair da Sala",
        mensagem: "Deseja sair da sala de aula?",
        botoes: [
          {
            mensagem: "Cancelar",
            evento: "ok",
          },
          {
            mensagem: "Sair",
            evento: "sairAula",
          }]
      };
    },
    redirecionar(atividade){
      if(atividade.idAtividade != 4)
      {
        if(!atividade.concluido)
          this.$router.push({ path: 'quiz', query: { codigo: atividade.idAtividade, id: atividade.id} });
        else
          this.msg = {
              visivel: true,
              titulo: "Ops...",
              mensagem: "Você já concluiu essa atividade e acertou "+atividade.nota+"%!",
              botoes: [{
                mensagem: "Ok",
                evento: "ok",
              }]
            };
      }
      else
        this.$router.push('/jogo');
    },
    entrarSala(cod) {
      this.$http.put("https://localhost:5001/api/usuarios/sala/" + cod, this.$session.get("usuario")).then(
        dados => {
          this.$session.set("idSala", dados.body.id);

          this.msg = {
            visivel: true,
            titulo: "Sucesso!",
            mensagem: "Você ingressou na sala: " + dados.body.nome,
            botoes: [{
              mensagem: "Legal",
              evento: "ok"
            }]
          };

          this.getSala(dados.body.id)
        },
        erro => {
          this.msg = {
            visivel: true,
            titulo: "Erro",
            mensagem: erro,
            botoes: [{
              mensagem: "Voltar",
              evento: "ok",
            }]
          };
        }
      );
    },
    getPostagens(idSala) {
      this.$http
        .get("https://localhost:5001/api/postagens/"+idSala) 
        .then( dados => {
            this.postagens = dados.body.reverse();

            this.postagens.map(p => {
              if(p.idAtividade != 4)
                 this.$http
                  .get("https://localhost:5001/api/usuarios/postagens/"+this.$session.get("usuario").id+"/"+p.id) 
                  .then(response => {
                    p.nota = response.body[0];
                    p.concluido = response.body[1];
                    this.$forceUpdate();
                  });
            });
          }
        ); 
    },
    getSala(id) {
      this.$http.get("https://localhost:5001/api/salas/id/"+id) 
        .then(
          dados => {
            this.sala = dados.body;
            this.prof = {
              id: this.sala.idProfessor,
              foto: "",
              nome:  this.sala.professor
            };
    
            this.$http.get("https://localhost:5001/api/usuarios/" + this.prof.id).then(response => this.prof.foto = response.body.foto);

            this.getPostagens(this.sala.id);
        });
    }
  },
  beforeCreate() {
    if (!this.$session.exists() || this.$session.get('MA'))
      this.$router.push('/');
  },
  created() {
    document.title = "Sala de Aula";

    if(this.$session.get("idSala") > 1)
      this.getSala(this.$session.get("idSala"));
  },
};
</script>

<style scoped>
.icone {
  width: 1.5em;
 margin-right: 05px;
}

.conc {
  background: rgb(119, 163, 103);
  background: rgb(103, 166, 65);
  width: fit-content;
  padding: 2px 10px;
  border-radius: 5px 5px 0 0;
  color: black; 
}

.n-conc {
  background: rgb(163, 103, 103);
  background: rgb(166, 65, 65);
  width: fit-content;
  padding: 2px 10px;
  border-radius: 5px 5px 0 0;
  color: black; 
}

.sala_vazia {
  display: flex;
  flex-direction: column-reverse;
  color: rgba(255, 255, 255, 0.604);
  justify-content: center;
  align-items: center;
  padding-top: 100px;
}

.centro{
  padding: 25px 15vw;
}

.inicio {
  display: flex;
  flex-direction:inherit;
  justify-content: center;
  align-items:center;
}

.quadrado{
  font-size: 3em;
  display: flex;
  flex-direction: column;
  box-sizing: border-box;
  padding: 20px;
  background: #f5f5f517;
  border-radius: 20px;
  color: whitesmoke;
  margin-right:5%;
}

img {
  width: 270px;
  border-radius: 20px;
  margin: auto;
}

input{
  border-radius: 10px;
  font-size:0.4em;
  text-align: center;
  border:transparent;
}

p {
  margin: 0;
  color:white;
  padding: 3px;
}

.p-else{
  font-size:0.6em;
  text-align: center;
  margin-bottom:3%;
}

.cod-sala{
  display: flex;
  flex-direction: row;
  height:15%;
  margin-top: 5%;
}

.cod-sala input{
  width:80%;
  margin-left: 5%;
}

button{
  font-size: 0.5em;
  border: none;
  color: white;
  background-color: rgba(241, 174, 30, 0.863);
  border-radius: 13px;
  cursor: pointer;
  font-weight: 1000;
  text-align: center;
  width:60%;
}

.botao {
  padding: 5px;
  margin: 5% auto 0; 
}

.cod-sala button{
  margin-left: 3%; 
}

.cima{
  width:80%;
  font-size:1.5em;
  margin-bottom:1%;
}

.infoSala {
  display: flex;
  justify-content: space-between;
}

.texto {
  margin-left: 2.3%;
  margin-bottom: 1%;
  margin-top:1%;
  color: darkgray;
}

.lista {
  display: flex;
  grid-gap: 15px;
  flex-direction: column;
  width: 80%;
}

</style>
