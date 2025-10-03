<script lang="ts">
	import { onMount } from 'svelte';
	import { goto } from '$app/navigation';
	import { authStore } from '$lib/stores/auth';
	import { getStudents, type Student } from '$lib/api/students';
	import {
		getLatestSensorData,
		type SensorData,
		createSensorData,
		type SensorDataFormData,
		getSensorData
	} from '$lib/api/sensor-data';
	import { getContainers, type Container } from '$lib/api/containers';
	import { createReport } from '$lib/api/reports';
	import {
		GraduationCap,
		ChevronRight,
		ChevronLeft,
		Activity,
		TrendingUp,
		Zap,
		Calendar,
		Thermometer,
		Cylinder
	} from 'lucide-svelte';
	import { toast } from 'svelte-sonner';
	import DataTable from '$lib/ui/datatable/datatable.svelte';
	import { useMobile } from '$lib/hooks/useMobile';

	// State
	let students = $state<Student[]>([]);
	let containers = $state<Container[]>([]);
	let latestSensorData = $state<SensorData | null>(null);
	let sensorHistory = $state<SensorData[]>([]);
	let selectedStudent = $state<Student | null>(null);
	let selectedContainer = $state<Container | null>(null);
	let sensorFormData = $state({
		temperature: null as number | null,
		humidity: null as number | null,
		gas: null as number | null,
		ph: null as number | null,
		status: '' as string
	});
	let loading = $state(false);
	let historyLoading = $state(false);
	let currentStep = $state(1);
	let isViewingHistory = $state(false);

	// Mobile detection
	const { isMobile } = useMobile(768);

	// Auth state
	let authState = $state($authStore);

	// Steps for the workflow
	const steps = [
		{ number: 1, label: 'Profil', active: true },
		{ number: 2, label: 'Drum', active: false },
		{ number: 3, label: 'Input THG', active: false },
		{ number: 4, label: 'Pilih Warna', active: false },
		{ number: 5, label: 'Riwayat Data', active: false }
	];

	// Status mapping for color selection
	const statusMapping = {
		green: 'sudah siap',
		yellow: 'hampir siap',
		red: 'belum siap'
	};

	// Determine LED status based on sensor readings
	const getLEDStatus = (sensorData: SensorData | null) => {
		if (!sensorData) return 'yellow'; // Default to yellow if no data

		const { temperature, humidity, ph, gas } = sensorData;

		// Simple logic for LED status based on sensor readings
		let warningCount = 0;
		let criticalCount = 0;

		// Temperature check (assuming normal range 15-30°C)
		if (temperature !== null && temperature !== undefined) {
			if (temperature < 10 || temperature > 35) criticalCount++;
			else if (temperature < 15 || temperature > 30) warningCount++;
		}

		// Humidity check (assuming normal range 40-70%)
		if (humidity !== null && humidity !== undefined) {
			if (humidity < 20 || humidity > 80) criticalCount++;
			else if (humidity < 40 || humidity > 70) warningCount++;
		}

		// pH check (assuming normal range 6-8)
		if (ph !== null && ph !== undefined) {
			if (ph < 5 || ph > 9) criticalCount++;
			else if (ph < 6 || ph > 8) warningCount++;
		}

		// Return status based on readings
		if (criticalCount > 0) return 'red';
		if (warningCount > 0) return 'yellow';
		return 'green';
	};

	// LED status based on sensor data - reactive
	let ledStatus = $state('yellow');

	// Update LED status when sensor data changes
	$effect(() => {
		ledStatus = getLEDStatus(latestSensorData);
	});

	onMount(() => {
		loadInitialData();
	});

	// Load students and container data
	const loadInitialData = async () => {
		loading = true;
		try {
			// Load students
			const studentsResponse = await getStudents({ limit: 50 });
			if (studentsResponse.data) {
				students = studentsResponse.data.students || [];
			}

			// Load containers for sensor data
			const containersResponse = await getContainers({ limit: 10 });
			if (containersResponse.data) {
				containers = containersResponse.data.containers || [];

				// Get latest sensor data from first container if available
				if (containers.length > 0) {
					const sensorResponse = await getLatestSensorData(containers[0].id);
					if (sensorResponse.data) {
						latestSensorData = sensorResponse.data;
					}
				}
			}
		} catch (error) {
			console.error('Failed to load initial data:', error);
			toast.error('Failed to load initial data');
		} finally {
			loading = false;
		}
	};

	// Handle student selection
	const selectStudent = (student: Student) => {
		selectedStudent = student;
	};

	// Handle container selection
	const selectContainer = (container: Container) => {
		selectedContainer = container;
	};

	// Navigate to login if not authenticated
	const goToLogin = () => {
		goto('/login');
	};

	const manageProfiles = () => {
		if (authState.isAuthenticated) {
			goto('/dashboard/students');
		} else {
			goToLogin();
		}
	};

	// Navigate to next step
	const goToStep = (step: number) => {
		if (step === 2 && !selectedStudent) {
			toast.error('Please select a student profile first');
			return;
		}
		if (step === 3 && !selectedContainer) {
			toast.error('Please select a container first');
			return;
		}
		currentStep = step;
	};

	// Navigate to drum selection
	const continueToDrum = () => {
		goToStep(2);
	};

	// Continue to THG input
	const continueToTHG = () => {
		goToStep(3);
	};

	// Continue to color selection
	const continueToColor = () => {
		goToStep(4);
	};

	// View history directly from step 1
	const viewHistory = () => {
		isViewingHistory = true;
		goToStep(5);
	};

	// Back navigation functions
	const goBackToProfile = () => {
		selectedContainer = null;
		sensorFormData = { temperature: null, humidity: null, gas: null, ph: null, status: '' };
		isViewingHistory = false;
		goToStep(1);
	};

	const goBackToDrum = () => {
		sensorFormData = { temperature: null, humidity: null, gas: null, ph: null, status: '' };
		isViewingHistory = false;
		goToStep(2);
	};

	const goBackToTHG = () => {
		isViewingHistory = false;
		goToStep(3);
	};

	// Reset to start fresh
	const startFresh = () => {
		selectedStudent = null;
		selectedContainer = null;
		sensorFormData = { temperature: null, humidity: null, gas: null, ph: null, status: '' };
		sensorHistory = [];
		isViewingHistory = false;
		goToStep(1);
	};

	// Submit sensor data with selected color/status
	const submitSensorData = async (selectedColor: string) => {
		if (!selectedContainer) {
			toast.error('No container selected');
			return;
		}

		const status = statusMapping[selectedColor as keyof typeof statusMapping];
		if (!status) {
			toast.error('Invalid color selection');
			return;
		}

		const submitData = {
			container_id: selectedContainer.id,
			temperature: sensorFormData.temperature,
			humidity: sensorFormData.humidity,
			gas: sensorFormData.gas,
			ph: sensorFormData.ph,
			status: status
		};

		// Create properly typed data object
		const filteredData: SensorDataFormData = {
			container_id: selectedContainer.id,
			status: status
		};

		if (sensorFormData.temperature !== null && sensorFormData.temperature !== undefined) {
			filteredData.temperature = sensorFormData.temperature;
		}
		if (sensorFormData.humidity !== null && sensorFormData.humidity !== undefined) {
			filteredData.humidity = sensorFormData.humidity;
		}
		if (sensorFormData.gas !== null && sensorFormData.gas !== undefined) {
			filteredData.gas = sensorFormData.gas;
		}
		if (sensorFormData.ph !== null && sensorFormData.ph !== undefined) {
			filteredData.ph = sensorFormData.ph;
		}

		try {
			const response = await createSensorData(filteredData);

			if (response.message) {
				// If sensor data was created successfully, also create a report
				if (selectedStudent && selectedContainer) {
					try {
						// Get the latest sensor data to obtain the ID for the report
						const latestResponse = await getLatestSensorData(selectedContainer.id);

						if (latestResponse.data) {
							const reportData = {
								student_id: selectedStudent.id,
								container_id: selectedContainer.id,
								sensor_data_id: latestResponse.data.id,
								notes: `Status: ${status} - Created via sensor monitoring interface`
							};

							await createReport(reportData);
							// No need to handle report response - just create it silently
						}
					} catch (reportError) {
						// Silent failure for report creation - don't show error to user
						console.error('Error creating report (background):', reportError);
					}
				}

				toast.success('Sensor data submitted successfully!');
				isViewingHistory = true; // Go directly to history mode
				await loadSensorHistory(); // Load the updated history
				goToStep(5);
			} else {
				toast.error(response.error || 'Failed to submit sensor data');
			}
		} catch (error) {
			console.error('Error submitting sensor data:', error);
			toast.error('Network error while submitting data');
		}
	};

	// Load sensor history for the selected container or all containers
	const loadSensorHistory = async () => {
		historyLoading = true;
		try {
			const params: any = {
				limit: 50 // Get more entries for general history view
			};

			// If we have a selected container, filter by it
			if (selectedContainer) {
				params.container_id = selectedContainer.id;
			}

			const response = await getSensorData(params);

			if (response.data) {
				sensorHistory = response.data.sensor_data || [];
			}
		} catch (error) {
			console.error('Error loading sensor history:', error);
			toast.error('Failed to load sensor history');
		} finally {
			historyLoading = false;
		}
	};

	// Load history when reaching step 5
	$effect(() => {
		if (currentStep === 5 && selectedStudent) {
			loadSensorHistory();
		}
	});

	// Define datatable columns for sensor history
	const historyColumns: Array<{
		key: string;
		label: string;
		align?: 'left' | 'right' | 'center';
		sortable?: boolean;
		sortType?: 'string' | 'number' | 'date';
		render?: (item: SensorData) => string;
	}> = [
		{
			key: 'created_at',
			label: 'Tanggal & Waktu',
			sortable: true,
			sortType: 'date' as const,
			render: (item: SensorData) => {
				const date = new Date(item.created_at);
				return `<div class="text-sm">
					<div class="font-medium">${date.toLocaleDateString('id-ID')}</div>
					<div class="text-surface-500 dark:text-surface-400">${date.toLocaleTimeString('id-ID')}</div>
				</div>`;
			}
		},
		{
			key: 'container_id',
			label: 'Container',
			align: 'center' as const,
			sortable: true,
			sortType: 'string' as const,
			render: (item: SensorData) => {
				const container = containers.find((c) => c.id === item.container_id);
				return container
					? `<span class="font-medium text-orange-600 dark:text-orange-400">${container.code}</span>`
					: `<span class="text-surface-400 text-xs">${item.container_id.slice(0, 8)}...</span>`;
			}
		},
		{
			key: 'temperature',
			label: 'Suhu (°C)',
			align: 'center' as const,
			sortable: true,
			sortType: 'number' as const,
			render: (item: SensorData) => {
				return item.temperature !== null && item.temperature !== undefined
					? `<span class="font-mono">${item.temperature}°C</span>`
					: '<span class="text-surface-400">-</span>';
			}
		},
		{
			key: 'humidity',
			label: 'Kelembaban (%)',
			align: 'center' as const,
			sortable: true,
			sortType: 'number' as const,
			render: (item: SensorData) => {
				return item.humidity !== null && item.humidity !== undefined
					? `<span class="font-mono">${item.humidity}%</span>`
					: '<span class="text-surface-400">-</span>';
			}
		},
		{
			key: 'gas',
			label: 'Gas',
			align: 'center' as const,
			sortable: true,
			sortType: 'number' as const,
			render: (item: SensorData) => {
				return item.gas !== null && item.gas !== undefined
					? `<span class="font-mono">${item.gas}</span>`
					: '<span class="text-surface-400">-</span>';
			}
		},
		{
			key: 'ph',
			label: 'pH',
			align: 'center' as const,
			sortable: true,
			sortType: 'number' as const,
			render: (item: SensorData) => {
				return item.ph !== null && item.ph !== undefined
					? `<span class="font-mono">${item.ph}</span>`
					: '<span class="text-surface-400">-</span>';
			}
		},
		{
			key: 'status',
			label: 'Status',
			align: 'center' as const,
			sortable: true,
			sortType: 'string' as const,
			render: (item: SensorData) => {
				const status = item.status || '';
				let badgeClass = 'px-2 py-1 rounded-full text-xs font-medium';

				switch (status.toLowerCase()) {
					case 'sudah siap':
						badgeClass += ' bg-green-100 text-green-800 dark:bg-green-900 dark:text-green-300';
						break;
					case 'hampir siap':
						badgeClass += ' bg-yellow-100 text-yellow-800 dark:bg-yellow-900 dark:text-yellow-300';
						break;
					case 'belum siap':
						badgeClass += ' bg-red-100 text-red-800 dark:bg-red-900 dark:text-red-300';
						break;
					default:
						badgeClass +=
							' bg-surface-100 text-surface-800 dark:bg-surface-700 dark:text-surface-300';
				}

				return status
					? `<span class="${badgeClass}">${status}</span>`
					: '<span class="text-surface-400">-</span>';
			}
		}
	];
</script>

<svelte:head>
	<title>Damayanti - Sensor Monitoring System</title>
</svelte:head>

<div
	class="min-h-screen bg-gradient-to-br from-blue-50 to-indigo-100 dark:from-surface-900 dark:to-surface-800"
>
	<div class="container mx-auto px-4 py-8 max-w-6xl">
		<!-- LED Status Information -->
		<div class="bg-white dark:bg-surface-800 rounded-2xl shadow-lg p-8 mb-8">
			<h1 class="text-2xl font-bold text-center text-surface-900 dark:text-surface-50 mb-8">
				Informasi Status Sensor LED
			</h1>

			<div class="flex justify-center items-center gap-16 mb-6">
				<!-- Red LED -->
				<div class="text-center">
					<div
						class="w-16 h-16 bg-red-500 rounded-full mb-3 mx-auto shadow-lg {ledStatus === 'red'
							? 'ring-4 ring-red-200 animate-pulse'
							: ''}"
					></div>
					<div class="text-red-600 font-bold text-lg">MERAH</div>
					<div class="text-surface-600 dark:text-surface-400 text-sm">Belum Siap</div>
				</div>

				<!-- Yellow LED -->
				<div class="text-center">
					<div
						class="w-16 h-16 bg-yellow-500 rounded-full mb-3 mx-auto shadow-lg {ledStatus ===
						'yellow'
							? 'ring-4 ring-yellow-200 animate-pulse'
							: ''}"
					></div>
					<div class="text-yellow-600 font-bold text-lg">KUNING</div>
					<div class="text-surface-600 dark:text-surface-400 text-sm">Hampir Siap</div>
				</div>

				<!-- Green LED -->
				<div class="text-center">
					<div
						class="w-16 h-16 bg-green-500 rounded-full mb-3 mx-auto shadow-lg {ledStatus === 'green'
							? 'ring-4 ring-green-200 animate-pulse'
							: ''}"
					></div>
					<div class="text-green-600 font-bold text-lg">HIJAU</div>
					<div class="text-surface-600 dark:text-surface-400 text-sm">Sudah Siap</div>
				</div>
			</div>

			<!-- Current Status -->
			{#if latestSensorData}
				<div class="bg-surface-100 dark:bg-surface-700 rounded-lg p-4 text-center">
					<div
						class="flex items-center justify-center gap-2 text-surface-700 dark:text-surface-300"
					>
						<Activity class="w-5 h-5" />
						<span class="font-medium">Status Terkini:</span>
						<span
							class="capitalize font-bold {ledStatus === 'green'
								? 'text-green-600'
								: ledStatus === 'yellow'
									? 'text-yellow-600'
									: 'text-red-600'}"
						>
							{ledStatus === 'green' ? 'Siap' : ledStatus === 'yellow' ? 'Persiapan' : 'Belum Siap'}
						</span>
					</div>
					{#if latestSensorData.temperature !== null}
						<div class="text-sm text-surface-600 dark:text-surface-400 mt-1">
							Suhu: {latestSensorData.temperature}°C
							{#if latestSensorData.humidity !== null}
								| Kelembaban: {latestSensorData.humidity}%
							{/if}
						</div>
					{/if}
				</div>
			{/if}
		</div>

		<!-- Workflow Steps -->
		<div class="bg-white dark:bg-surface-800 rounded-2xl shadow-lg p-4 md:p-8 mb-8">
			<h3
				class="text-center text-base md:text-lg font-semibold text-surface-900 dark:text-surface-50 mb-4 md:mb-6"
			>
				Langkah-langkah Monitoring Sensor
			</h3>
			<div class="flex justify-center items-center gap-1 md:gap-2 overflow-x-auto pb-2">
				{#each steps as step, index}
					<div class="flex items-center">
						<div class="flex flex-col items-center text-center min-w-[80px] md:min-w-[120px]">
							<div
								class="w-8 h-8 md:w-12 md:h-12 rounded-full flex items-center justify-center mb-2 md:mb-3 text-sm md:text-lg font-bold {step.number ===
								currentStep
									? 'bg-blue-500 text-white shadow-lg'
									: step.number < currentStep
										? 'bg-green-500 text-white'
										: 'bg-surface-200 dark:bg-surface-600 text-surface-600 dark:text-surface-400'}"
							>
								{#if step.number < currentStep}
									✓
								{:else}
									{step.number}
								{/if}
							</div>
							<span
								class="font-medium text-xs md:text-sm {step.number === currentStep
									? 'text-blue-600 dark:text-blue-400'
									: step.number < currentStep
										? 'text-green-600 dark:text-green-400'
										: 'text-surface-600 dark:text-surface-400'} leading-tight"
							>
								{$isMobile ? step.label.split(' ')[0] : step.label}
							</span>
						</div>
						{#if index < steps.length - 1}
							<div
								class="w-4 md:w-8 h-0.5 {step.number < currentStep
									? 'bg-green-400'
									: 'bg-surface-300 dark:bg-surface-600'} mx-2 md:mx-4 mt-[-10px] md:mt-[-20px]"
							></div>
						{/if}
					</div>
				{/each}
			</div>
		</div>

		<!-- Main Content -->
		<div class="bg-white dark:bg-surface-800 rounded-2xl shadow-lg p-4 md:p-8">
			<!-- Step 1: Student Profile Selection -->
			{#if currentStep === 1}
				<!-- Header -->
				<div class="text-center mb-8">
					<h2 class="text-3xl font-bold text-surface-900 dark:text-surface-50 mb-4">
						Pilih Profil Siswa
					</h2>
					<p class="text-surface-600 dark:text-surface-400 text-lg mb-6">
						Klik pada profil siswa untuk memulai monitoring sensor
					</p>
				</div>

				<!-- Student Profiles Grid -->
				{#if loading}
					<div class="flex justify-center items-center py-12">
						<div class="text-center">
							<div
								class="animate-spin rounded-full h-12 w-12 border-b-2 border-primary-600 mx-auto mb-4"
							></div>
							<p class="text-surface-600 dark:text-surface-400">Memuat profil siswa...</p>
						</div>
					</div>
				{:else if students.length === 0}
					<div class="text-center py-12">
						<GraduationCap class="w-16 h-16 text-surface-400 mx-auto mb-4" />
						<h3 class="text-lg font-medium text-surface-900 dark:text-surface-50 mb-2">
							Belum ada profil siswa
						</h3>
						<p class="text-surface-600 dark:text-surface-400 mb-4">
							Silakan tambahkan profil siswa terlebih dahulu
						</p>
						{#if authState.isAuthenticated}
							<button class="btn preset-filled-primary-500" onclick={manageProfiles}>
								Tambah Profil Siswa
							</button>
						{:else}
							<button class="btn preset-filled-primary-500" onclick={goToLogin}>
								Login untuk Mengelola
							</button>
						{/if}
					</div>
				{:else}
					<div class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 xl:grid-cols-5 gap-4 md:gap-6">
						{#each students as student}
							<button
								class="group p-4 md:p-6 rounded-xl border-2 transition-all duration-200 hover:scale-105 {selectedStudent?.id ===
								student.id
									? 'border-primary-500 bg-primary-50 dark:bg-primary-900/20 shadow-lg'
									: 'border-surface-200 dark:border-surface-600 bg-surface-50 dark:bg-surface-700 hover:border-primary-300 hover:shadow-md'}"
								onclick={() => selectStudent(student)}
							>
								<div class="text-center">
									<div
										class="w-12 h-12 md:w-16 md:h-16 mx-auto mb-2 md:mb-3 rounded-full bg-gradient-to-br from-blue-500 to-purple-600 flex items-center justify-center shadow-lg group-hover:shadow-xl transition-shadow"
									>
										<GraduationCap class="w-6 h-6 md:w-8 md:h-8 text-white" />
									</div>
									<h3
										class="font-bold text-surface-900 dark:text-surface-50 text-xs md:text-sm leading-tight"
									>
										{$isMobile
											? student.full_name.split(' ').slice(0, 2).join(' ')
											: student.full_name}
									</h3>
									{#if selectedStudent?.id === student.id}
										<div
											class="mt-1 md:mt-2 text-xs text-primary-600 dark:text-primary-400 font-medium"
										>
											✓ Terpilih
										</div>
									{/if}
								</div>
							</button>
						{/each}
					</div>

					<!-- Selected Student Info -->
					{#if selectedStudent}
						<div
							class="mt-6 md:mt-8 p-3 md:p-4 bg-primary-50 dark:bg-primary-900/20 rounded-lg border border-primary-200 dark:border-primary-800"
						>
							<div class="flex flex-col sm:flex-row sm:items-center sm:justify-between gap-4">
								<div class="flex items-center gap-3">
									<div
										class="w-8 h-8 md:w-10 md:h-10 rounded-full bg-primary-500 flex items-center justify-center"
									>
										<GraduationCap class="w-4 h-4 md:w-5 md:h-5 text-white" />
									</div>
									<div>
										<h4
											class="font-bold text-surface-900 dark:text-surface-50 text-sm md:text-base"
										>
											Profil yang dipilih: {$isMobile
												? selectedStudent.full_name.split(' ').slice(0, 2).join(' ')
												: selectedStudent.full_name}
										</h4>
										<p class="text-xs md:text-sm text-surface-600 dark:text-surface-400">
											Bergabung sejak {new Date(selectedStudent.created_at).toLocaleDateString(
												'id-ID'
											)}
										</p>
									</div>
								</div>
								<div class="flex flex-col sm:flex-row gap-2 md:gap-3">
									<button
										class="btn preset-outlined-primary-500 hover:scale-105 flex items-center justify-center gap-2 text-sm md:text-base"
										onclick={viewHistory}
									>
										<Calendar class="w-3 h-3 md:w-4 md:h-4" />
										{$isMobile ? 'Riwayat' : 'Lihat Riwayat'}
									</button>
									<button
										class="btn preset-filled-primary-500 hover:scale-105 flex items-center justify-center gap-2 text-sm md:text-base"
										onclick={continueToDrum}
									>
										<TrendingUp class="w-3 h-3 md:w-4 md:h-4" />
										{$isMobile ? 'Container' : 'Pilih Container'}
									</button>
								</div>
							</div>
						</div>
					{/if}
				{/if}

				<!-- Step 2: Container/Drum Selection -->
			{:else if currentStep === 2}
				<div class="text-center mb-8">
					<h2 class="text-2xl font-bold text-surface-900 dark:text-surface-50 mb-2">
						Pilih Container / Drum
					</h2>
					<p class="text-surface-600 dark:text-surface-400">
						Siswa: <strong>{selectedStudent?.full_name}</strong>
					</p>
				</div>

				{#if containers.length === 0}
					<div class="text-center py-12">
						<Activity class="w-16 h-16 text-surface-400 mx-auto mb-4" />
						<h3 class="text-lg font-medium text-surface-900 dark:text-surface-50 mb-2">
							Belum ada container tersedia
						</h3>
					</div>
				{:else}
					<div class="grid grid-cols-2 sm:grid-cols-3 lg:grid-cols-4 gap-4 md:gap-6 mb-6 md:mb-8">
						{#each containers as container}
							<button
								class="group p-4 md:p-6 rounded-xl border-2 transition-all duration-200 hover:scale-105 {selectedContainer?.id ===
								container.id
									? 'border-blue-500 bg-blue-50 dark:bg-blue-900/20 shadow-lg'
									: 'border-surface-200 dark:border-surface-600 bg-surface-50 dark:bg-surface-700 hover:border-blue-300 hover:shadow-md'}"
								onclick={() => selectContainer(container)}
							>
								<div class="text-center">
									<div
										class="w-12 h-12 md:w-16 md:h-16 mx-auto mb-2 md:mb-3 rounded-full bg-gradient-to-br from-orange-500 to-red-600 flex items-center justify-center shadow-lg"
									>
										<Cylinder class="w-6 h-6 md:w-8 md:h-8 text-white" />
									</div>
									<h3 class="font-bold text-surface-900 dark:text-surface-50 text-xs md:text-sm">
										{container.code}
									</h3>
									{#if selectedContainer?.id === container.id}
										<div class="mt-1 md:mt-2 text-xs text-blue-600 dark:text-blue-400 font-medium">
											✓ Terpilih
										</div>
									{/if}
								</div>
							</button>
						{/each}
					</div>

					{#if selectedContainer}
						<div class="flex justify-center gap-4">
							<button
								class="btn preset-outlined-surface-600 hover:scale-105 flex items-center gap-2"
								onclick={goBackToProfile}
							>
								<ChevronLeft class="w-4 h-4" />
								Kembali ke Profil
							</button>
							<button
								class="btn preset-filled-primary-500 hover:scale-105 flex items-center gap-2"
								onclick={continueToTHG}
							>
								Lanjut ke Input THG
								<ChevronRight class="w-4 h-4" />
							</button>
						</div>
					{:else}
						<div class="flex justify-center">
							<button
								class="btn preset-outlined-surface-600 hover:scale-105 flex items-center gap-2"
								onclick={goBackToProfile}
							>
								<ChevronLeft class="w-4 h-4" />
								Kembali ke Profil
							</button>
						</div>
					{/if}
				{/if}

				<!-- Step 3: THG Input -->
			{:else if currentStep === 3}
				<div class="text-center mb-8">
					<h2 class="text-2xl font-bold text-surface-900 dark:text-surface-50 mb-2">
						Input Data Sensor (THG)
					</h2>
					<p class="text-surface-600 dark:text-surface-400 mb-2">
						Container: <strong>{selectedContainer?.code}</strong> | Siswa:
						<strong>{selectedStudent?.full_name}</strong>
					</p>
					<p class="text-surface-500 dark:text-surface-400 text-lg">
						✨ Semua field bersifat opsional - Isi sesuai kebutuhan
					</p>
				</div>

				<div class="max-w-4xl mx-auto px-2 md:px-0">
					<div class="grid grid-cols-1 md:grid-cols-2 gap-4 md:gap-8">
						<!-- Temperature -->
						<div
							class="bg-gradient-to-br from-red-50 to-orange-50 dark:from-red-900/20 dark:to-orange-900/20 border-2 border-red-200 dark:border-red-700 p-6 rounded-xl shadow-lg"
						>
							<div class="flex items-center gap-2 sm:gap-3 mb-3 sm:mb-4">
								<div
									class="w-10 h-10 sm:w-12 sm:h-12 bg-red-500 rounded-full flex items-center justify-center"
								>
									<Thermometer class="w-4 h-4 sm:w-6 sm:h-6 text-white" />
								</div>
								<div>
									<label
										for="temperature"
										class="block text-base sm:text-lg font-bold text-red-700 dark:text-red-300"
									>
										Suhu (°C)
									</label>
									<span class="text-red-500 dark:text-red-400 text-xs sm:text-sm"
										>Temperature - opsional</span
									>
								</div>
							</div>
							<input
								id="temperature"
								type="number"
								step="0.1"
								bind:value={sensorFormData.temperature}
								class="input w-full text-base sm:text-lg h-10 sm:h-12 placeholder:text-surface-200 dark:placeholder:text-surface-700 border-red-300 focus:border-red-500 focus:ring-red-500"
								placeholder="Contoh: 25.5"
							/>
						</div>

						<!-- Humidity -->
						<div
							class="bg-gradient-to-br from-blue-50 to-cyan-50 dark:from-blue-900/20 dark:to-cyan-900/20 border-2 border-blue-200 dark:border-blue-700 p-4 sm:p-6 rounded-xl shadow-lg"
						>
							<div class="flex items-center gap-2 sm:gap-3 mb-3 sm:mb-4">
								<div
									class="w-10 h-10 sm:w-12 sm:h-12 bg-blue-500 rounded-full flex items-center justify-center"
								>
									<Zap class="w-4 h-4 sm:w-6 sm:h-6 text-white" />
								</div>
								<div>
									<label
										for="humidity"
										class="block text-base sm:text-lg font-bold text-blue-700 dark:text-blue-300"
									>
										Kelembaban (%)
									</label>
									<span class="text-blue-500 dark:text-blue-400 text-xs sm:text-sm"
										>Humidity - opsional</span
									>
								</div>
							</div>
							<input
								id="humidity"
								type="number"
								step="0.1"
								min="0"
								max="100"
								bind:value={sensorFormData.humidity}
								class="input w-full text-base sm:text-lg h-10 sm:h-12 placeholder:text-surface-200 dark:placeholder:text-surface-700 border-blue-300 focus:border-blue-500 focus:ring-blue-500"
								placeholder="Contoh: 65.2"
							/>
						</div>

						<!-- Gas -->
						<div
							class="bg-gradient-to-br from-purple-50 to-indigo-50 dark:from-purple-900/20 dark:to-indigo-900/20 border-2 border-purple-200 dark:border-purple-700 p-4 sm:p-6 rounded-xl shadow-lg"
						>
							<div class="flex items-center gap-2 sm:gap-3 mb-3 sm:mb-4">
								<div
									class="w-10 h-10 sm:w-12 sm:h-12 bg-purple-500 rounded-full flex items-center justify-center"
								>
									<Activity class="w-4 h-4 sm:w-6 sm:h-6 text-white" />
								</div>
								<div>
									<label
										for="gas"
										class="block text-base sm:text-lg font-bold text-purple-700 dark:text-purple-300"
									>
										Gas Reading
									</label>
									<span class="text-purple-500 dark:text-purple-400 text-xs sm:text-sm"
										>Gas Level - opsional</span
									>
								</div>
							</div>
							<input
								id="gas"
								type="number"
								step="0.1"
								min="0"
								bind:value={sensorFormData.gas}
								class="input w-full text-base sm:text-lg h-10 sm:h-12 placeholder:text-surface-200 dark:placeholder:text-surface-700 border-purple-300 focus:border-purple-500 focus:ring-purple-500"
								placeholder="Contoh: 450"
							/>
						</div>

						<!-- pH -->
						<div
							class="bg-gradient-to-br from-green-50 to-emerald-50 dark:from-green-900/20 dark:to-emerald-900/20 border-2 border-green-200 dark:border-green-700 p-4 sm:p-6 rounded-xl shadow-lg"
						>
							<div class="flex items-center gap-2 sm:gap-3 mb-3 sm:mb-4">
								<div
									class="w-10 h-10 sm:w-12 sm:h-12 bg-green-500 rounded-full flex items-center justify-center"
								>
									<TrendingUp class="w-4 h-4 sm:w-6 sm:h-6 text-white" />
								</div>
								<div>
									<label
										for="ph"
										class="block text-base sm:text-lg font-bold text-green-700 dark:text-green-300"
									>
										pH Level
									</label>
									<span class="text-green-500 dark:text-green-400 text-xs sm:text-sm"
										>Acidity Level - opsional</span
									>
								</div>
							</div>
							<input
								id="ph"
								type="number"
								step="0.1"
								min="0"
								max="14"
								bind:value={sensorFormData.ph}
								class="input w-full text-base sm:text-lg h-10 sm:h-12 placeholder:text-surface-200 dark:placeholder:text-surface-700 border-green-300 focus:border-green-500 focus:ring-green-500"
								placeholder="Contoh: 7.2"
							/>
						</div>
					</div>

					<div class="flex justify-center gap-2 sm:gap-4 mt-6 sm:mt-8">
						<button
							class="btn preset-outlined-surface-600 hover:scale-105 flex items-center gap-1 sm:gap-2 text-sm sm:text-base px-3 sm:px-4 py-2"
							onclick={goBackToDrum}
						>
							<ChevronLeft class="w-3 h-3 sm:w-4 sm:h-4" />
							{$isMobile ? 'Container' : 'Kembali ke Container'}
						</button>
						<button
							class="btn preset-filled-primary-500 hover:scale-105 flex items-center gap-1 sm:gap-2 text-sm sm:text-base px-3 sm:px-4 py-2"
							onclick={continueToColor}
						>
							{$isMobile ? 'Pilih Warna' : 'Lanjut ke Pilih Warna'}
							<ChevronRight class="w-3 h-3 sm:w-4 sm:h-4" />
						</button>
					</div>
				</div>

				<!-- Step 4: Color/Status Selection -->
			{:else if currentStep === 4}
				<div class="text-center mb-6 sm:mb-8">
					<h2 class="text-xl sm:text-2xl font-bold text-surface-900 dark:text-surface-50 mb-2">
						Pilih Status Warna
					</h2>
					<p class="text-surface-600 dark:text-surface-400 mb-3 sm:mb-4 text-sm sm:text-base">
						Pilih warna sesuai dengan kondisi sensor
					</p>
					<p class="text-surface-500 dark:text-surface-400 text-xs sm:text-sm">
						Container: <strong>{selectedContainer?.code}</strong> | Siswa:
						<strong>{selectedStudent?.full_name}</strong>
					</p>
				</div>

				<!-- Back Button -->
				<div class="flex justify-center mb-4 sm:mb-6">
					<button
						class="btn preset-outlined-surface-600 hover:scale-105 flex items-center gap-1 sm:gap-2 text-sm sm:text-base px-3 sm:px-4 py-2"
						onclick={goBackToTHG}
					>
						<ChevronLeft class="w-3 h-3 sm:w-4 sm:h-4" />
						{$isMobile ? 'Input Data' : 'Kembali ke Input Data'}
					</button>
				</div>

				<div class="grid grid-cols-1 md:grid-cols-3 gap-4 sm:gap-8 mb-6 sm:mb-8 max-w-4xl mx-auto">
					<!-- Red Status -->
					<button
						class="group text-center p-4 sm:p-8 rounded-2xl border-4 border-red-200 hover:border-red-400 hover:scale-105 transition-all bg-white dark:bg-surface-800 shadow-lg"
						onclick={() => submitSensorData('red')}
					>
						<div
							class="w-16 h-16 sm:w-24 sm:h-24 bg-red-500 rounded-full mb-3 sm:mb-6 mx-auto shadow-lg group-hover:shadow-xl group-hover:bg-red-600 transition-all"
						></div>
						<div class="text-red-600 font-bold text-lg sm:text-2xl mb-1 sm:mb-2">MERAH</div>
						<div class="text-surface-600 dark:text-surface-400 text-base sm:text-lg font-medium">
							Belum Siap
						</div>
					</button>

					<!-- Yellow Status -->
					<button
						class="group text-center p-4 sm:p-8 rounded-2xl border-4 border-yellow-200 hover:border-yellow-400 hover:scale-105 transition-all bg-white dark:bg-surface-800 shadow-lg"
						onclick={() => submitSensorData('yellow')}
					>
						<div
							class="w-16 h-16 sm:w-24 sm:h-24 bg-yellow-500 rounded-full mb-3 sm:mb-6 mx-auto shadow-lg group-hover:shadow-xl group-hover:bg-yellow-600 transition-all"
						></div>
						<div class="text-yellow-600 font-bold text-lg sm:text-2xl mb-1 sm:mb-2">KUNING</div>
						<div class="text-surface-600 dark:text-surface-400 text-base sm:text-lg font-medium">
							Hampir Siap
						</div>
					</button>

					<!-- Green Status -->
					<button
						class="group text-center p-4 sm:p-8 rounded-2xl border-4 border-green-200 hover:border-green-400 hover:scale-105 transition-all bg-white dark:bg-surface-800 shadow-lg"
						onclick={() => submitSensorData('green')}
					>
						<div
							class="w-16 h-16 sm:w-24 sm:h-24 bg-green-500 rounded-full mb-3 sm:mb-6 mx-auto shadow-lg group-hover:shadow-xl group-hover:bg-green-600 transition-all"
						></div>
						<div class="text-green-600 font-bold text-lg sm:text-2xl mb-1 sm:mb-2">HIJAU</div>
						<div class="text-surface-600 dark:text-surface-400 text-base sm:text-lg font-medium">
							Sudah Siap
						</div>
					</button>
				</div>

				<!-- Data Summary -->
				<div class="max-w-2xl mx-auto bg-surface-50 dark:bg-surface-700 rounded-lg p-6">
					<h4
						class="font-medium text-surface-900 dark:text-surface-50 mb-3 sm:mb-4 text-sm sm:text-base"
					>
						Ringkasan Data:
					</h4>
					<div class="grid grid-cols-1 sm:grid-cols-2 gap-2 sm:gap-4 text-xs sm:text-sm">
						<div>Siswa: <strong>{selectedStudent?.full_name}</strong></div>
						<div>Container: <strong>{selectedContainer?.code}</strong></div>
						{#if sensorFormData.temperature !== null}
							<div>Suhu: <strong>{sensorFormData.temperature}°C</strong></div>
						{/if}
						{#if sensorFormData.humidity !== null}
							<div>Kelembaban: <strong>{sensorFormData.humidity}%</strong></div>
						{/if}
						{#if sensorFormData.gas !== null}
							<div>Gas: <strong>{sensorFormData.gas}</strong></div>
						{/if}
						{#if sensorFormData.ph !== null}
							<div>pH: <strong>{sensorFormData.ph}</strong></div>
						{/if}
					</div>
				</div>

				<!-- Step 5: History -->
			{:else if currentStep === 5}
				<!-- History View Header -->
				<div class="text-center mb-6 sm:mb-8">
					<div class="flex items-center justify-center gap-2 sm:gap-3 mb-3 sm:mb-4">
						<Calendar class="w-8 h-8 sm:w-12 sm:h-12 text-primary-600" />
						<h2 class="text-xl sm:text-2xl font-bold text-surface-900 dark:text-surface-50">
							Riwayat Data Sensor
						</h2>
					</div>
					{#if selectedStudent}
						<p class="text-surface-600 dark:text-surface-400 mb-4 sm:mb-6 text-sm sm:text-base">
							Siswa: <strong>{selectedStudent?.full_name}</strong>
						</p>
					{/if}
					<div class="flex flex-col sm:flex-row justify-center gap-3 sm:gap-4 px-4 sm:px-0">
						<button
							class="btn preset-outlined-surface-600 hover:scale-105 flex items-center justify-center gap-1 sm:gap-2 text-base sm:text-lg px-4 sm:px-6 py-2 sm:py-3"
							onclick={() => {
								currentStep = 1;
								sensorHistory = [];
								isViewingHistory = false;
							}}
						>
							<ChevronLeft class="w-4 h-4 sm:w-5 sm:h-5" />
							{$isMobile ? 'Profil' : 'Kembali ke Profil'}
						</button>
						<button
							class="btn preset-filled-primary-500 hover:scale-105 flex items-center justify-center gap-1 sm:gap-2 text-base sm:text-lg px-4 sm:px-6 py-2 sm:py-3"
							onclick={() => {
								isViewingHistory = false;
								continueToDrum();
							}}
						>
							<TrendingUp class="w-4 h-4 sm:w-5 sm:h-5" />
							{$isMobile ? 'Data Baru' : 'Input Data Baru'}
						</button>
					</div>
				</div>

				<!-- Sensor History Table -->
				<div class="mt-6 sm:mt-8">
					<div
						class="flex flex-col sm:flex-row sm:items-center gap-2 sm:gap-3 mb-4 sm:mb-6 px-4 sm:px-0"
					>
						<div class="flex items-center gap-2 sm:gap-3">
							<Calendar class="w-5 h-5 sm:w-6 sm:h-6 text-primary-600" />
							<h3 class="text-lg sm:text-xl font-bold text-surface-900 dark:text-surface-50">
								{selectedContainer ? 'Data Container' : 'Semua Data Sensor'}
							</h3>
						</div>
						{#if selectedContainer}
							<span class="text-xs sm:text-sm text-surface-500 dark:text-surface-400">
								Container: {selectedContainer?.code}
							</span>
						{/if}
					</div>

					<div class="overflow-x-auto -mx-4 sm:mx-0">
						<div class="min-w-full px-4 sm:px-0">
							<DataTable
								data={sensorHistory}
								columns={historyColumns}
								loading={historyLoading}
								emptyIcon={Thermometer}
								emptyTitle="Belum ada data sensor"
								emptyDescription="Data sensor untuk container ini akan muncul di sini setelah ada data yang diinput"
								keyField="id"
								defaultSort={{ column: 'created_at', direction: 'desc' }}
							/>
						</div>
					</div>
				</div>
			{/if}
		</div>
	</div>
</div>
