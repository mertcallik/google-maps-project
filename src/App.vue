<script setup>
import GoogleMap from './components/GoogleMap.vue';
</script>

<template>
  <div class="min-h-screen bg-gray-100">
    <main class="container mx-auto px-4 py-6">
      <div class="bg-white rounded-lg shadow-lg overflow-hidden">
        <div class="p-4 border-b">
          <h2 class="text-xl font-semibold">Map</h2>
          <p class="text-gray-600 text-sm">Click on any point on the map to create a route. You can drag or remove markers.</p>
        </div>
        
        <GoogleMap />
      </div>

      <div class="mt-6 bg-white rounded-lg shadow-lg p-6">
        <h2 class="text-xl font-semibold mb-4">How to Use?</h2>
        
        <ul class="list-disc pl-6 space-y-2 text-gray-700">
          <li>Click on the map to mark a point.</li>
          <li>You can change marker positions by dragging them.</li>
          <li>When you click on a marker, detailed information is displayed with a removal option.</li>
          <li>When two or more points are marked, a line is created between them and the distance is calculated.</li>
          <li>You can use the "Clear All Markers" button to remove all points.</li>
        </ul>
      </div>
    </main>

    <footer class="bg-gray-800 text-white py-4 mt-8">
      <div class="container mx-auto px-4 text-center">
        <p>© Made by iDIGITEK with ❤️</p>
      </div>
    </footer>
  </div>
</template>

<style>
/* Global styles */
</style>
