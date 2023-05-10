<template>
    <div class="container mx-auto px-4 py-6">
        <h1 class="text-2xl font-semibold mb-4">Historique des recherches</h1>

        <div class="mb-4">
            <input type="text"
                class="border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline"
                placeholder="Rechercher par date, référence, marque ou modèle" v-model="searchQuery"
                @input="filterResults" />
        </div>

        <div class="table-responsive">
            <table class="table-auto w-full mb-6">
                <thead class="rounded-t-xl">
                    <tr class="bg-gray-800">
                        <th class="text-white px-4 py-2">Date</th>
                        <th class="text-white px-4 py-2">Référence</th>
                        <th class="text-white px-4 py-2">Marque</th>
                        <th class="text-white px-4 py-2">Modèle</th>
                        <th class="text-white px-4 py-2">Version</th>
                        <th class="text-white px-4 py-2">Années</th>
                        <th class="text-white px-4 py-2">Kilomètres</th>
                        <th class="text-white px-4 py-2">Prix</th>
                        <th class="text-white px-4 py-2">Actions</th>
                    </tr>
                </thead>
                <tbody class="shadow-xl">
                    <tr v-for="(search, index) in filteredSearchHistory" :key="index">
                        <td class="border px-4 py-2 text-sm">{{ formatDate(search.createdAt) }}</td>
                        <td class="border px-4 py-2 text-sm">{{ search.ref }}</td>
                        <td class="border px-4 py-2 text-sm">
                            {{ formatArray(search.dataSearch.map((item) => item.make.toUpperCase())) }}
                        </td>
                        <td class="border px-4 py-2 text-sm">
                            {{ formatArray(search.dataSearch.map((item) => item.model.toUpperCase())) }}
                        </td>
                        <td class="border px-4 py-2 text-sm">
                            {{ formatArray(search.dataSearch.map((item) => item.keywords)) }}
                        </td>
                        <td class="border px-4 py-2 text-sm">
                            {{ formatArray(search.dataSearch.map((item) => item.year)) }}
                        </td>
                        <td class="border px-4 py-2 text-sm">
                            {{ formatArray(search.dataSearch.map((item) => item.mileage)) }} km
                        </td>
                        <td class="border px-4 py-2 text-sm">{{ formatPrice(search.dataSearch) }}</td>
                        <td class="border px-4 py-2 text-sm">
                            <button @click="deleteSearchHistory(search.id)"><i class="fas fa-trash"></i></button>
                        </td>
                    </tr>
                </tbody>
            </table>
        </div>

        <div>
            <!-- Ajoutez ici la pagination en utilisant une bibliothèque de votre choix ou en créant votre propre solution. -->
        </div>
    </div>
</template>

<style scoped>
.table-responsive {
    display: block;
    width: 100%;
    overflow-x: auto;
    -webkit-overflow-scrolling: touch;
}

@media screen and (min-width: 640px) {
    .table-responsive {
        display: table;
    }
}
</style>
  
<script>
import { ref, computed } from 'vue';
import { Auth, API, graphqlOperation } from 'aws-amplify';
import { listSearchHistories } from '../../graphql/queries';
import { deleteSearchHistory as deleteSearchHistoryMutation } from '../../graphql/mutations';

export default {
    setup() {

        const searchHistory = ref([]);
        const searchQuery = ref('');

        // Ajoutez cette méthode dans votre composant
        const formatDate = (dateString) => {
            const date = new Date(dateString);
            const day = String(date.getDate()).padStart(2, '0');
            const month = String(date.getMonth() + 1).padStart(2, '0');
            const year = date.getFullYear();
            return `${day}/${month}/${year}`;
        };

        const formatArray = (array) => {
            if (!array) return 'x';
            if (array.length <= 3) return array.join(', ');
            return `${array.slice(0, 3).join(', ')}...`;
        };

        const formatPrice = (dataSearch) => {
            if (dataSearch.length === 1) return dataSearch[0].price;
            const total = dataSearch.reduce((sum, item) => sum + item.price, 0);
            return (total / dataSearch.length).toFixed(2);
        };

        const filteredSearchHistory = computed(() => {
            if (!searchQuery.value) {
                return searchHistory.value;
            }
            return searchHistory.value.filter((search) => {
                const query = searchQuery.value.toLowerCase();
                return (
                    search.createdAt.toLowerCase().includes(query) ||
                    search.ref.toLowerCase().includes(query) ||
                    search.dataSearch.make.toLowerCase().includes(query) ||
                    search.dataSearch.model.toLowerCase().includes(query)
                );
            });
        });

        const fetchSearchHistory = async () => {
            try {
                const currentUser = await Auth.currentAuthenticatedUser();
                const { data } = await API.graphql({
                    query: listSearchHistories,
                    authMode: 'AMAZON_COGNITO_USER_POOLS',
                });

                searchHistory.value = data.listSearchHistories.items
                    .filter((item) => item.owner === currentUser.username)
                    .sort((a, b) => new Date(b.createdAt) - new Date(a.createdAt))
                    .map(search => {
                        return {
                            ...search,
                            dataSearch: JSON.parse(search.dataSearch),
                        };
                    });

                console.log(searchHistory.value);
            } catch (error) {
                console.error('Error fetching search history:', error);
            }
        };

        const deleteSearchHistory = async (searchId) => {
            try {
                await API.graphql({
                    query: deleteSearchHistoryMutation,
                    variables: { input: { id: searchId } },
                    authMode: 'AMAZON_COGNITO_USER_POOLS',
                });
                fetchSearchHistory();
                alert('Suppression réussie!');
            } catch (error) {
                alert('Erreur lors de la suppression!');
            }
        };

        fetchSearchHistory();

        const filterResults = () => {
            // The search will be performed by the computed property filteredSearchHistory.
        };

        return {
            formatDate,
            formatArray,
            formatPrice,
            searchHistory,
            searchQuery,
            filteredSearchHistory,
            deleteSearchHistory,
            filterResults,
        };
    },
};
</script>

  