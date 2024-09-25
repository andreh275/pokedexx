<template>
  <div id="app">

    <div class="youtube-audio-container">
      <iframe
        src="https://youtu.be/PMqmYvl1L24?si=1Tz967ppVmW8b5S8"
        frameborder="0"
        allow="autoplay"
        allowfullscreen
        style="position: absolute; top: -9999px; width: 1px; height: 1px;"
      ></iframe>
    </div>
    <header class="pokedex-header">
      <div class="header-logo">
        <h1>Pokedex</h1>
      </div>
      <div class="search-container">
        <button @click="showRandomPokemon">Sorpréndeme</button>
        <input
          v-model="searchQuery"
          @input="filterPokemons"
          placeholder="Buscar Pokémon por nombre o número..."
        />
        <button @click="filterPokemons">Buscar</button>
      </div>
    </header>

    <!-- Mensaje si no se encuentra el Pokémon -->
    <div v-if="!selectedPokemon && searchQuery">
      <div class="not-found">
        <h2>Pokémon no encontrado</h2>
        <img src="https://raw.githubusercontent.com/PokeAPI/sprites/master/sprites/pokemon/other/official-artwork/54.png" alt="Pokémon confundido" />
      </div>
    </div>

    <!-- Detalles del Pokémon seleccionado -->
    <div v-if="selectedPokemon" class="pokemon-list">
      <div 
        class="pokemon-card" 
        :class="{ legendary: selectedPokemon.isLegendary }"
        :style="getPokemonCardStyle(selectedPokemon)"
      >
        <h3>{{ selectedPokemon.name }}</h3>
        <p class="id-number">Nº {{ selectedPokemon.id }}</p>
        <img :src="selectedPokemon.sprites.front_default" :alt="selectedPokemon.name" />
        <div class="types">
          <h4>Tipos del Pokémon:</h4>
          <div v-for="(type, index) in selectedPokemon.types" :key="index" class="type" :style="{ backgroundColor: getTypeColor(type.type.name) }">
            {{ type.type.name }}
          </div>
        </div>

        <div class="weaknesses">
          <h4>Debilidades del Pokémon:</h4>
          <div v-for="(weakness, index) in getWeaknesses(selectedPokemon)" :key="index" class="weakness" :style="{ backgroundColor: getTypeColor(weakness) }">
            {{ weakness }}
          </div>
        </div>

        <div class="stats">
          <h4>Estadísticas:</h4>
          <div v-for="(stat, index) in selectedPokemon.stats" :key="index" class="stat">
            <span>{{ statNames[index] }}: {{ stat.base_stat }}/255</span>
            <div class="stat-bar">
              <div class="bar-fill" :style="{ width: (stat.base_stat / 255) * 100 + '%' }"></div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      pokemons: [],
      selectedPokemon: null,
      searchQuery: '',
      statNames: ['Hp', 'Attack', 'Defense', 'Special-attack', 'Special-defense', 'Speed'],
      typeWeaknesses: {}, // Se llenará con datos de las debilidades originales
    };
  },
  methods: {
    async fetchPokemons() {
      try {
        const ids = Array.from({ length: 1025 }, (_, index) => index + 1);
        const promises = ids.map(id => fetch(`https://pokeapi.co/api/v2/pokemon/${id}`).then(res => res.json()));
        this.pokemons = await Promise.all(promises);

        // Cargar las debilidades de los tipos
        const types = await Promise.all(
          Array.from(new Set(this.pokemons.flatMap(pokemon => pokemon.types.map(type => type.type.name)))).map(type => fetch(`https://pokeapi.co/api/v2/type/${type}`).then(res => res.json()))
        );

        types.forEach(type => {
          this.typeWeaknesses[type.name] = type.damage_relations.double_damage_from.map(weakness => weakness.name);
        });

        // Mostrar un Pokémon aleatorio al cargar la página
        this.showRandomPokemon();
      } catch (error) {
        console.error('Error al cargar los Pokémon:', error);
      }
    },
    showRandomPokemon() {
      const randomIndex = Math.floor(Math.random() * this.pokemons.length);
      this.selectedPokemon = this.pokemons[randomIndex];
    },
    filterPokemons() {
      const query = this.searchQuery.toLowerCase();

      if (!isNaN(query)) {
        const pokemonById = this.pokemons.find(pokemon => pokemon.id === parseInt(query));
        this.selectedPokemon = pokemonById || null;
      } 
      // Si no es un número, busca por nombre
      else {
        const pokemonByName = this.pokemons.find(pokemon =>
          pokemon.name.toLowerCase().includes(query)
        );
        this.selectedPokemon = pokemonByName || null;
      }
    },
    getTypeColor(type) {
      const colors = {
        normal: '#A8A878',
        fire: '#F08030',
        water: '#6890F0',
        electric: '#F8D030',
        grass: '#78C850',
        ice: '#98D8D8',
        fighting: '#C03028',
        poison: '#A040A0',
        ground: '#E0C068',
        flying: '#A890F0',
        psychic: '#F85888',
        bug: '#A8B820',
        rock: '#B8A038',
        ghost: '#705898',
        dragon: '#7038F8',
        dark: '#705848',
        steel: '#B8B8D0',
        fairy: '#F0B6BC'
      };
      return colors[type] || '#D0D0D0';
    },
    getLegendaryColor(pokemon) {
      // Definir colores especiales para Pokémon legendarios según su importancia
      const legendaryColors = {
        'legendary1': '#FFD700', // Dorado
        'legendary2': '#C0C0C0', // Plateado
        'legendary3': '#D3D3D3', // Tornasol
      };
      return legendaryColors[pokemon.legendaryStatus] || '#D0D0D0';
    },
    getPokemonCardStyle(pokemon) {
      if (pokemon.isLegendary) {
        return {
          '--legendary-color': this.getLegendaryColor(pokemon),
          backgroundColor: 'var(--legendary-color)',
          animation: 'legendaryAnimation 1.5s infinite'
        };
      } else {
        const primaryType = pokemon.types[0].type.name;
        const secondaryType = pokemon.types[1] ? pokemon.types[1].type.name : null;
        return {
          background: secondaryType
            ? `linear-gradient(to right, ${this.getTypeColor(primaryType)}, ${this.getTypeColor(secondaryType)})`
            : this.getTypeColor(primaryType),
          animation: 'pokemonAnimation 1.5s infinite'
        };
      }
    },
    getWeaknesses(pokemon) {
      const weaknesses = new Set();
      pokemon.types.forEach(type => {
        const typeName = type.type.name;
        if (this.typeWeaknesses[typeName]) {
          this.typeWeaknesses[typeName].forEach(weakness => weaknesses.add(weakness));
        }
      });
      return Array.from(weaknesses);
    }
  },
  mounted() {
    this.fetchPokemons();
  }
};
</script>

<style>
/* Estilos generales del app */
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  text-align: center;
  background-color: #f4f4f4;
  color: #333;
  padding: 20px;
  box-sizing: border-box;
}

/* Contenedor de las tarjetas de Pokémon */
.pokemon-list {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 80vh;
}

/* Estilos generales para las tarjetas de Pokémon */
.pokemon-card {
  color: #333;
  padding: 20px;
  border-radius: 10px;
  transition: transform 0.3s ease, background 0.5s ease;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
  max-width: 300px;
  text-align: center;
  background: linear-gradient(to right, var(--type-color1, #fff), var(--type-color2, #fff));
}

/* Estilos para las imágenes grandes de los Pokémon */
.pokemon-image {
  max-width: 100%;
  height: auto;
}

/* Estilos para las barras de estadísticas */
.stat-bar {
  width: 100%;
  height: 10px;
  background-color: #ddd;
  position: relative;
  border-radius: 5px;
}

.bar-fill {
  height: 100%;
  background-color: #4CAF50;
  border-radius: 5px;
}

/* Estilos para los tipos de Pokémon */
.type, .weakness {
  display: inline-block;
  padding: 5px 10px;
  margin: 5px;
  color: #fff;
  border-radius: 15px;
  text-transform: capitalize;
}

/* Estilos para el encabezado de la Pokedex */
.pokedex-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
  background-color: #333;
  color: #fff;
  padding: 10px;
  border-radius: 10px;
}

.header-logo {
  font-size: 1.5em;
}

/* Estilos del contenedor de búsqueda */
.search-container {
  display: flex;
  align-items: center;
}

.search-container input {
  padding: 8px;
  font-size: 1em;
  border: none;
  border-radius: 5px;
  margin-left: 10px;
}

.search-container button {
  background-color: #4CAF50;
  color: white;
  padding: 10px 15px;
  margin-left: 10px;
  border: none;
  border-radius: 5px;
  cursor: pointer;
}

.search-container button:hover {
  background-color: #45a049;
}

.youtube-audio-container {
  position: absolute;
  top: -9999px;
  width: 1px;
  height: 1px;
}

/* Estilos para cuando el Pokémon es legendario */
.legendary {
  background-color: var(--legendary-color);
  animation: legendaryAnimation 1.5s infinite;
}

/* Animaciones */
@keyframes pokemonAnimation {
  0% { transform: translateY(0); }
  50% { transform: translateY(-10px); }
  100% { transform: translateY(0); }
}

@keyframes legendaryAnimation {
  0% { transform: scale(1); }
  50% { transform: scale(1.05); }
  100% { transform: scale(1); }
}

</style>
