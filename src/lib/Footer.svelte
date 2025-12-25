<footer id="contact">
	<div class="footer-content">
		<div class="contact-section">
			<span class="section-label">Data Period</span>
			<a>Feb 2020 – Nov 2025 (70 Months)</a>
		</div>
	</div>

	<div class="footer-bottom">
		<span>Miriam Abbas</span>
		<span>Information Design Winter Semester 2025/26</span>
	</div>
</footer>

<style>
	footer {
		padding: 3rem 1.5rem;
		border-top: 1px solid rgba(255, 139, 76, 0.2);
		opacity: 0.7;
	}

	.footer-content {
		display: flex;
		flex-direction: column;
		gap: 2rem;
	}

	.contact-section {
		display: flex;
		flex-direction: column;
		gap: 1rem;
	}

	.section-label {
        
		font-size: 0.6rem;
		letter-spacing: 0.1em;
		text-transform: uppercase;
		display: block;
	}

	.footer-bottom {
		margin-top: 4rem;
		padding-top: 1.5rem;
		display: flex;
		flex-direction: column;
		gap: 1rem;
		font-size: 0.6rem;
		letter-spacing: 0.1em;
		text-transform: uppercase;
	}

	@media (min-width: 768px) {
		footer {
            margin-top: 10rem;
			padding: 4rem 3rem;
		}

		.section-label {
			font-size: 0.8rem;
		}

		.footer-content {
			flex-direction: row;
			align-items: flex-end;
			justify-content: space-between;
		}


		.footer-bottom {
            font-size: 0.8rem;
			margin-top: 6rem;
			flex-direction: row;
			align-items: center;
			justify-content: space-between;
		}
	}
</style>
