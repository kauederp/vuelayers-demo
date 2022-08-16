<template #default="popup">
    <section class="card">
    <header class="card-header">
        <p class="card-header-title">
        Feature ID {{ feature.id }}
        </p>
        <a class="card-header-icon" title="Close"
            @click="selectedFeatures = selectedFeatures.filter(f => f.id !== feature.id)">
        <b-icon icon="close"></b-icon>
        </a>
    </header>
    <div class="card-content">
        <div class="content">
        <p>
            Overlay popup content for Feature with ID <strong>{{ feature.id }}</strong>
        </p>
        <p>
            Popup: {{ JSON.stringify(popup) }}
        </p>
        <p>
            Feature: {{ JSON.stringify({id: feature.id, properties: feature.properties}) }}
        </p>
        </div>
    </div>
    </section>
</template>