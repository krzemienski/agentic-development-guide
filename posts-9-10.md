================================================================================
POST 9 OF 10: FROM GITHUB REPOS TO AUDIO STORIES
Agentic Development: 10 Lessons from 8,481 AI Coding Sessions
================================================================================

What if you could listen to a codebase?

Not read documentation. Not scan source files. Listen -- like a podcast episode about FastAPI's architecture, or a nature documentary narrating how Django evolved in the wild.

That is exactly what code-tales does. Give it a GitHub URL, pick one of 9 narrative styles, and in under two minutes you get a fully synthesized audio story about the repository. Clone, analyze, narrate with Claude, synthesize with ElevenLabs. Four pipeline stages. One command.

  code-tales generate --repo https://github.com/tiangolo/fastapi --style documentary

This is post 9 of 10 in the Agentic Development series. The companion repo is at github.com/krzemienski/code-tales. Everything I quote here is real code you can read, run, and extend.


THE PIPELINE ARCHITECTURE

The core abstraction is a four-stage pipeline orchestrated by a single class. Here is the actual orchestrator:

  # From: src/code_tales/pipeline/orchestrate.py

  class CodeTalesPipeline:
      """Orchestrates the full code-tales pipeline.

      Stages:
        1. Resolve repository (clone URL or validate local path)
        2. Analyze repository structure and metadata
        3. Generate narration script via Claude
        4. Synthesize audio via ElevenLabs (or save text only)
      """

      def generate(
          self,
          repo_url_or_path: str,
          style_name: str,
          output_dir: Optional[Path] = None,
      ) -> AudioOutput:

The generate method runs all four stages sequentially with Rich progress bars showing real-time status. The key design decision: every stage is independently importable. You can call analyze_repository without cloning, or generate_script without synthesizing audio. The pipeline composes; it does not mandate.

[DIAGRAM: Four-stage pipeline flow -- Input (GitHub URL or local path) -> Clone (gitpython shallow depth=1) -> Analyze (languages, dependencies, frameworks, patterns) -> Narrate (Claude streaming API) -> Synthesize (ElevenLabs TTS) -> Output (script.md + story.mp3)]

The pipeline resolution logic handles both remote URLs and local paths:

  # From: src/code_tales/pipeline/orchestrate.py

  def _resolve_repo(self, repo_url_or_path: str) -> tuple[Path, Optional[Path]]:
      if repo_url_or_path.startswith(("http://", "https://", "git@")):
          temp_dir = Path(tempfile.mkdtemp(prefix="code-tales-"))
          repo_path = clone_repository(
              url=repo_url_or_path,
              target_dir=temp_dir,
              depth=self.config.clone_depth,
          )
          return repo_path, temp_dir

      # Local path
      local = Path(repo_url_or_path).expanduser().resolve()
      if not (local / ".git").exists():
          raise ValueError(
              f"Local path is not a git repository (no .git dir): {local}"
          )
      return local, None

Shallow clones (depth=1) are the default. We need the file tree, not the full commit history. This keeps clone times under a few seconds for most repositories.


THE ANALYSIS ENGINE

Before Claude ever sees a repository, the analysis stage extracts structured metadata. This is where code-tales distinguishes itself from "just paste the README into Claude." The analyzer detects languages by file extension counts, extracts dependencies from 8 different package manager formats, identifies frameworks, and recognizes architectural patterns.

The dependency extraction alone covers package.json, requirements.txt, pyproject.toml, Cargo.toml, go.mod, Gemfile, Package.swift, and pom.xml:

  # From: src/code_tales/pipeline/analyze.py

  def _extract_dependencies(repo_path: Path) -> list[Dependency]:
      deps: list[Dependency] = []

      # package.json (npm/yarn)
      pkg_json = repo_path / "package.json"
      if pkg_json.exists():
          data = json.loads(pkg_json.read_text(encoding="utf-8"))
          for section in ("dependencies", "devDependencies", "peerDependencies"):
              for name, version in (data.get(section) or {}).items():
                  deps.append(
                      Dependency(name=name, version=str(version), source="package.json")
                  )

      # ... repeated for requirements.txt, pyproject.toml, Cargo.toml,
      #     go.mod, Gemfile, Package.swift, pom.xml

The pattern detection identifies monorepos, microservices, REST APIs, GraphQL schemas, CLI tools, web applications, mobile apps, MVC architectures, test coverage, CI/CD pipelines, and containerization. All heuristic-based, all derived from directory structure and file presence -- no LLM calls required.

  # From: src/code_tales/pipeline/analyze.py

  def _detect_patterns(repo_path: Path) -> list[str]:
      patterns: list[str] = []

      # Monorepo detection
      workspace_files = [
          repo_path / "pnpm-workspace.yaml",
          repo_path / "lerna.json",
          repo_path / "nx.json",
          repo_path / "rush.json",
      ]
      if any(f.exists() for f in workspace_files):
          patterns.append("Monorepo")

This matters because the structured analysis gives Claude specific, quantified data to work with -- not just "here's a bunch of source code, figure it out." The prompt includes file counts, language percentages, dependency lists, detected frameworks, and architectural patterns. Claude narrates from data, not from guesswork.

All of this feeds into a Pydantic model that constrains the data shape throughout:

  # From: src/code_tales/models.py

  class RepoAnalysis(BaseModel):
      name: str
      description: str = ""
      languages: list[Language] = Field(default_factory=list)
      dependencies: list[Dependency] = Field(default_factory=list)
      file_tree: str = ""
      key_files: list[FileInfo] = Field(default_factory=list)
      total_files: int = 0
      total_lines: int = 0
      frameworks: list[str] = Field(default_factory=list)
      patterns: list[str] = Field(default_factory=list)
      readme_content: Optional[str] = None


NINE STYLES, NINE DIFFERENT EXPERIENCES

This is the feature that makes code-tales more than a utility. Each of the 9 narrative styles is defined as a YAML configuration with its own tone directive, section structure template, ElevenLabs voice ID, and voice synthesis parameters. The same repository analyzed in "documentary" style and "fiction" style produces fundamentally different audio experiences.

Here is the documentary style -- think David Attenborough observing code as a living ecosystem:

  # From: src/code_tales/styles/documentary.yaml

  name: documentary
  description: A nature documentary-style narration that observes the codebase
    as a living ecosystem, in the tradition of David Attenborough.
  tone: Measured, observational, and quietly awestruck. Speak as if witnessing
    remarkable natural phenomena. Use present tense for immediacy.
  voice_id: "onwK4e9ZLuTAKqWW03F9"
  voice_params:
    stability: 0.7
    similarity_boost: 0.8
    style: 0.3
  example_opener: |
    Here we observe a remarkable specimen of software engineering,
    quietly going about its work in the digital wilderness...

Now compare that to the fiction style -- literary drama with characters and conflict:

  # From: src/code_tales/styles/fiction.yaml

  name: fiction
  description: A literary fiction narrative that transforms code into a
    dramatic story with characters, conflict, and emotional arc.
  tone: Lyrical, evocative, and dramatic. Write as if describing a novel's
    setting and characters. Use vivid metaphors and literary devices.
  voice_id: "EXAVITQu4vr4xnSDxMaL"
  voice_params:
    stability: 0.3
    similarity_boost: 0.7
    style: 0.8
  example_opener: |
    In the vast digital landscape, there existed a repository that dared
    to dream in a language most could not speak...

Notice the voice parameter differences. Documentary uses high stability (0.7) and low style (0.3) -- controlled, authoritative delivery. Fiction drops stability to 0.3 and cranks style to 0.8 -- more expressive, more emotional range. These are not arbitrary numbers. They map directly to ElevenLabs' voice synthesis parameters that control how much freedom the TTS model has in its delivery.

The full roster: debate (structured argument for and against design decisions), documentary, executive (crisp leadership briefing), fiction, interview (Q&A with an expert), podcast (casual tech talk with hot takes), storytelling (hero's journey), technical (dense engineering review), and tutorial (educational walkthrough).

The podcast style captures a completely different register:

  # From: src/code_tales/styles/podcast.yaml

  name: podcast
  tone: Casual, enthusiastic, and opinionated. Sound like you've actually
    used this project and have genuine thoughts. Use filler phrases
    naturally. Share hot takes.
  voice_params:
    stability: 0.5
    similarity_boost: 0.7
    style: 0.6
  example_opener: |
    Hey everyone, welcome back! So I've been digging into this really
    interesting repo this week and honestly, I had to record an
    episode about it...

Each style also defines a structure_template that guides Claude's section organization. The executive style forces "Executive Summary, Key Metrics, Architecture Overview, Risk Assessment, Recommendations." The storytelling style uses "Once Upon a Time, The Call to Adventure, The Journey, The Breakthrough, The Legacy." Same data, completely different narrative architecture.


THE NARRATION ENGINE

The narration stage builds a detailed prompt from the analysis data and sends it to Claude's streaming API. The system message sets the narrative voice, and the user message provides the full repository analysis:

  # From: src/code_tales/pipeline/narrate.py

  with client.messages.stream(
      model=config.claude_model,
      max_tokens=config.max_script_tokens,
      system=system_message,
      messages=[{"role": "user", "content": prompt}],
  ) as stream:
      raw_text = stream.get_final_message().content[0].text

Streaming prevents HTTP timeouts on long scripts. The system message includes explicit formatting rules for audio output -- no bullet points, no tables, natural speech patterns, contractions, flowing sentences. Critical for TTS quality: the text Claude generates will be spoken aloud, not read on a screen.

The parser then splits Claude's response on markdown headings, extracts voice directions from bracketed text (like "[speak slowly]" or "[with enthusiasm]"), and strips formatting artifacts:

  # From: src/code_tales/pipeline/narrate.py

  def _clean_content(content: str) -> str:
      # Remove bold/italic markers
      content = re.sub(r"\*\*([^*]+)\*\*", r"\1", content)
      content = re.sub(r"\*([^*]+)\*", r"\1", content)
      # Remove inline code backticks
      content = re.sub(r"`([^`]+)`", r"\1", content)
      # Remove code blocks
      content = re.sub(r"```[\s\S]*?```", "", content)
      return content.strip()

This is one of those details that seems trivial but matters enormously. If you leave markdown formatting in the text that goes to TTS, the synthesized audio sounds wrong. Asterisks create unnatural pauses. Backticks confuse the speech model. The cleaning step bridges two fundamentally different output modalities.


THE AUDIO SYNTHESIS LAYER

ElevenLabs integration includes retry logic with exponential backoff, per-style voice selection, and graceful degradation to text-only output:

  # From: src/code_tales/pipeline/synthesize.py

  payload = {
      "text": text,
      "model_id": "eleven_multilingual_v2",
      "voice_settings": {
          "stability": voice_params.get("stability", 0.5),
          "similarity_boost": voice_params.get("similarity_boost", 0.75),
          "style": voice_params.get("style", 0.0),
          "use_speaker_boost": True,
      },
      "output_format": "mp3_44100_128",
  }

  for attempt in range(_MAX_RETRIES):
      with httpx.Client(timeout=120.0) as client:
          response = client.post(url, json=payload, headers=headers)
      if response.status_code == 200:
          return response.content
      if response.status_code == 429:
          delay = _RETRY_BASE_DELAY * (2 ** attempt)
          time.sleep(delay)
          continue

The text output is always generated regardless of whether audio synthesis succeeds. If you do not have an ElevenLabs API key, you still get a complete narration script as markdown.


THE BUILD STORY

Code-tales was built using the parallel AI development patterns described earlier in this series. 636 commits across 90 worktree branches. 91 specification documents. 37 validation gates.

The audio debugging saga deserves special mention: 9 commits to fix race conditions in the synthesis pipeline. The problem was subtle -- when sections were concatenated for TTS, the natural pause markers between sections were being dropped during the markdown cleaning step. This produced audio where one section ran directly into the next without any breathing room. The fix was to build the TTS text separately from the display text, preserving paragraph breaks as natural pauses:

  # From: src/code_tales/pipeline/synthesize.py

  _PAUSE_BETWEEN_SECTIONS = "\n\n"  # Natural paragraph break for TTS

  def _build_tts_text(script: NarrationScript) -> str:
      parts: list[str] = []
      for section in script.sections:
          parts.append(section.heading + ".")
          parts.append(section.content)
      return _PAUSE_BETWEEN_SECTIONS.join(parts)

Simple in retrospect. Nine commits to arrive at two newline characters. That is debugging audio pipelines for you.


THE CLI EXPERIENCE

The user-facing interface is a Click CLI with three commands -- generate, preview, and list-styles:

  # From: src/code_tales/cli.py

  @cli.command()
  @click.option("--repo", "repo_url", help="GitHub repository URL.")
  @click.option("--path", "local_path", type=click.Path(exists=True, path_type=Path))
  @click.option("--style", "-s", required=True)
  @click.option("--output", "-o", "output_dir", type=click.Path(path_type=Path))
  @click.option("--no-audio", is_flag=True, default=False)
  def generate(ctx, repo_url, local_path, style, output_dir, no_audio):
      pipeline = CodeTalesPipeline(config=config)
      result = pipeline.generate(
          repo_url_or_path=target,
          style_name=style,
          output_dir=out_dir,
      )

All configuration flows through environment variables via a Pydantic config model:

  # From: src/code_tales/config.py

  class CodeTalesConfig(BaseModel):
      anthropic_api_key: Optional[str] = None
      elevenlabs_api_key: Optional[str] = None
      output_dir: Path = Field(default_factory=lambda: Path("./output"))
      clone_depth: int = 1
      max_files_to_analyze: int = 100
      max_file_size_bytes: int = 100_000
      claude_model: str = "claude-sonnet-4-6"
      max_script_tokens: int = 4096

The style system is extensible via YAML. Drop a new .yaml file in the styles directory -- or load one from any path at runtime -- and you have a new narrative voice. The README includes a full example of creating a "noir detective" style. The style registry handles loading, validation, and lookup:

  # From: src/code_tales/styles/registry.py

  class StyleRegistry:
      def load_builtin_styles(self) -> dict[str, StyleConfig]:
          yaml_files = sorted(_STYLES_DIR.glob("*.yaml"))
          for yaml_file in yaml_files:
              style = self._load_yaml(yaml_file)
              self._styles[style.name] = style
          return self._styles

      def get_style(self, name: str) -> StyleConfig:
          if name not in self._styles:
              available = ", ".join(sorted(self._styles.keys()))
              raise KeyError(
                  f"Style '{name}' not found. Available styles: {available}"
              )
          return self._styles[name]


WHAT I LEARNED

The most counterintuitive lesson: the analysis stage matters more than the narration stage. Give Claude mediocre data and you get mediocre narration regardless of how good the style prompt is. Give Claude precise, quantified analysis -- "47.3% Python, 12 dependencies including FastAPI and Pydantic, REST API + CLI Tool patterns detected, 23,000 lines across 145 files" -- and the narration practically writes itself.

The second lesson: voice parameters are a design surface. Changing ElevenLabs stability from 0.7 to 0.3 transforms a documentary into a dramatic reading. These three numbers -- stability, similarity_boost, style -- are the audio equivalent of typography choices. They deserve the same attention.

The third lesson: text-to-speech requires a different writing style than text-to-screen. Every markdown artifact that slips through the cleaning stage is audible. Every bullet point that should have been a sentence creates an awkward pause. When you build a pipeline that crosses modalities, the boundary between them is where the bugs live.

Companion repo: github.com/krzemienski/code-tales

#AgenticDevelopment #ClaudeAI #ElevenLabs #TextToSpeech #DeveloperTools


================================================================================
POST 10 OF 10: 21 AI-GENERATED SCREENS, ZERO FIGMA FILES
Agentic Development: 10 Lessons from 8,481 AI Coding Sessions
================================================================================

I described an entire web application in plain English. The AI generated 21 production screens -- with components, design tokens, and validation -- in a single session. No Figma. No hand-written CSS. No designer in the loop.

This is not a prototype. It is a complete brutalist-cyberpunk web application with 47 design tokens, 5 component primitives, 4 example compositions, and a 107-action Puppeteer validation suite that proves every screen renders correctly.

This is post 10 of 10 in the Agentic Development series. The companion repo is at github.com/krzemienski/stitch-design-to-code. Everything I quote here is real code from that repo.


THE WORKFLOW

The cycle looks deceptively simple:

1. Craft a text prompt with the design system embedded
2. Feed it to Stitch MCP (an AI design generation tool)
3. Review the visual output for spec compliance
4. Convert to React components with data-testid attributes
5. Run Puppeteer validation against all 21 screens
6. If anything fails, go back to step 1

[DIAGRAM: Circular workflow -- Craft Prompt (embed design system, describe layout + states) -> Stitch MCP (generate visual design from structured prompt) -> Design Output (review for spec compliance, check brand + colors) -> React Conversion (build components, add data-testid attrs) -> Puppeteer Validation (107 actions / 21 screens) -> All Checks Pass? If yes, ship. If no, loop back to Craft Prompt.]

The critical insight, learned the hard way: the prompt must contain the complete design system every single time. Not "see previous specs." Not "same as before." The full token set, repeated verbatim, in every prompt for every screen. Context window drift is real, and it manifests as subtle visual inconsistencies across screens generated in the same session.


THE DESIGN TOKEN SYSTEM

Every visual decision in the application is derived from 47 tokens defined in a single JSON file. Here is the brutalist-cyberpunk palette:

  // From: design-system/tokens.json

  {
    "colors": {
      "background": "#000000",
      "primary": "#e050b0",
      "secondary": "#4dacde",
      "surface": "#111111",
      "surfaceAlt": "#1a1a1a",
      "surfaceHover": "#222222",
      "text": "#ffffff",
      "textSecondary": "#a0a0a0",
      "textMuted": "#606060",
      "border": "#2a2a2a",
      "borderAccent": "#3a3a3a",
      "success": "#22c55e",
      "warning": "#f59e0b",
      "error": "#ef4444",
      "primaryGlow": "rgba(224, 80, 176, 0.3)",
      "secondaryGlow": "rgba(77, 172, 222, 0.3)"
    }
  }

Pure black background. Hot pink primary accent. Cyan secondary. No gray area -- literally. The surface colors step from #111111 to #1a1a1a to #222222 in tight increments, creating depth through value rather than hue.

But the most distinctive design decision lives in the border radius tokens:

  // From: design-system/tokens.json

  "borderRadius": {
    "none": "0px",
    "sm": "0px",
    "base": "0px",
    "md": "0px",
    "lg": "0px",
    "xl": "0px",
    "2xl": "0px",
    "full": "0px"
  }

Every single border radius value is 0px. Not "mostly square with some rounding." Zero. Everywhere. This is not laziness -- it is a deliberate brutalist design choice. When components from shadcn/ui reference borderRadius.md or borderRadius.lg, they resolve to 0px. The token system enforces the aesthetic at the configuration layer, not the component layer.

Typography is equally opinionated. One font family everywhere:

  // From: design-system/tokens.json

  "typography": {
    "fontFamily": "JetBrains Mono, monospace",
    "fontFamilyFallback": "Courier New, Courier, monospace"
  }

JetBrains Mono for headlines, body copy, buttons, labels, navigation -- everything. Combined with the 0px border radius, this produces an aggressive terminal-aesthetic that looks like nothing else in the React ecosystem.

These tokens flow into Tailwind via a preset that maps every token to a utility class:

  // From: design-system/tailwind-preset.js

  const tokens = require('./tokens.json');

  const preset = {
    theme: {
      extend: {
        colors: {
          background: tokens.colors.background,
          primary: {
            DEFAULT: tokens.colors.primary,
            glow: tokens.colors.primaryGlow,
            subtle: tokens.colors.primarySubtle,
          },
          secondary: {
            DEFAULT: tokens.colors.secondary,
            glow: tokens.colors.secondaryGlow,
            subtle: tokens.colors.secondarySubtle,
          },
          surface: {
            DEFAULT: tokens.colors.surface,
            alt: tokens.colors.surfaceAlt,
            hover: tokens.colors.surfaceHover,
          },
        },
        fontFamily: {
          mono: [tokens.typography.fontFamily, tokens.typography.fontFamilyFallback],
          sans: [tokens.typography.fontFamily, tokens.typography.fontFamilyFallback],
        },
        borderRadius: {
          none: tokens.borderRadius.none,
          sm: tokens.borderRadius.sm,
          DEFAULT: tokens.borderRadius.base,
          md: tokens.borderRadius.md,
          lg: tokens.borderRadius.lg,
          xl: tokens.borderRadius.xl,
          '2xl': tokens.borderRadius['2xl'],
          full: tokens.borderRadius.full,
        },
      },
    },
  };

Notice: fontFamily.sans maps to JetBrains Mono. Even when components use Tailwind's font-sans class, they get the monospaced font. The design system is inescapable.

The preset also defines custom background gradients that use the primary and secondary colors at low opacity:

  // From: design-system/tailwind-preset.js

  backgroundImage: {
    'primary-gradient': `linear-gradient(135deg, ${tokens.colors.primary}20 0%, transparent 60%)`,
    'hero-gradient': `radial-gradient(ellipse at top, ${tokens.colors.primary}15 0%, transparent 60%), radial-gradient(ellipse at bottom right, ${tokens.colors.secondary}10 0%, transparent 50%)`,
    'grid-pattern': `linear-gradient(${tokens.colors.border} 1px, transparent 1px), linear-gradient(90deg, ${tokens.colors.border} 1px, transparent 1px)`,
  },

That grid-pattern utility creates a subtle wire-grid background that reinforces the terminal aesthetic. One line in the config, reusable across every screen.


THE COMPONENT LAYER

The Button component demonstrates how CVA (Class Variance Authority) combines with the token system to produce a flexible, type-safe component API. Eight variants, eight sizes, forwardRef, Radix Slot for composition:

  // From: components/ui/button.tsx

  const buttonVariants = cva(
    // Base styles
    [
      'inline-flex items-center justify-center gap-2',
      'font-mono text-sm font-semibold',
      'border-0 outline-none',
      'transition-all duration-150',
      'cursor-pointer select-none',
      'disabled:pointer-events-none disabled:opacity-50',
      'focus-visible:ring-2 focus-visible:ring-primary focus-visible:ring-offset-2 focus-visible:ring-offset-background',
      // Brutalist — no border radius anywhere
      'rounded-none',
    ].join(' '),
    {
      variants: {
        variant: {
          /** Hot pink fill — primary CTA */
          default: [
            'bg-primary text-black',
            'hover:bg-[#c040a0] hover:shadow-[0_0_20px_rgba(224,80,176,0.4)]',
            'active:scale-[0.98] active:bg-[#b03090]',
          ].join(' '),

          /** Cyan outline */
          'secondary-outline': [
            'border border-secondary text-secondary bg-transparent',
            'hover:bg-secondary hover:text-black hover:shadow-[0_0_16px_rgba(77,172,222,0.3)]',
            'active:scale-[0.98]',
          ].join(' '),

          /** Ghost — subtle surface hover */
          ghost: [
            'bg-transparent text-text-secondary',
            'hover:bg-surface hover:text-text',
            'active:scale-[0.98]',
          ].join(' '),
        },

        size: {
          xs: 'h-7 px-2 text-xs',
          sm: 'h-8 px-3 text-xs',
          default: 'h-10 px-4 text-sm',
          lg: 'h-12 px-6 text-base',
          xl: 'h-14 px-8 text-lg',
          icon: 'h-10 w-10 p-0',
          'icon-sm': 'h-8 w-8 p-0',
          'icon-lg': 'h-12 w-12 p-0',
        },
      },
    }
  );

The hover effects are where the brutalist aesthetic comes alive. The primary button gets a hot pink glow on hover -- shadow-[0_0_20px_rgba(224,80,176,0.4)]. The secondary-outline variant gets a cyan glow. Combined with the sharp 0px corners and JetBrains Mono text, these glow effects create a cyberpunk UI that feels like interacting with a terminal from the future.

The component supports Radix Slot for composition, a loading state with an inline spinner, and proper aria-disabled handling:

  // From: components/ui/button.tsx

  const Button = React.forwardRef<HTMLButtonElement, ButtonProps>(
    ({ className, variant, size, asChild = false, loading = false, children, disabled, ...props }, ref) => {
      const Comp = asChild ? Slot : 'button';

      return (
        <Comp
          ref={ref}
          className={cn(buttonVariants({ variant, size }), className)}
          disabled={disabled || loading}
          aria-disabled={disabled || loading}
          {...props}
        >
          {loading ? (
            <>
              <LoadingSpinner />
              <span className="opacity-70">{children}</span>
            </>
          ) : (
            children
          )}
        </Comp>
      );
    }
  );

This is one button component. There are 5 total component primitives (Button, Card, Input, Tabs, Badge) and 4 example compositions (HomeHero, ResourceCard, AuthForm, AdminTabs). From these 9 building blocks, 21 screens are constructed.


THE VALIDATION SUITE

This is where most AI design workflows fall apart. You generate beautiful screens, manually eyeball them, declare victory, and ship something that breaks in production. The stitch-design-to-code approach replaces eyeballing with 107 programmatic Puppeteer actions across all 21 screens.

The checks are defined declaratively:

  // From: validation/puppeteer-checks.js

  {
    name: 'home-design-tokens',
    route: '/',
    actions: [
      { type: 'navigate', expected: 200 },
      {
        type: 'evaluate',
        expected: () => {
          const body = document.body;
          const bg = window.getComputedStyle(body).backgroundColor;
          return bg === 'rgb(0, 0, 0)';
        },
      },
    ],
  },

That check validates that the home page background is literally rgb(0, 0, 0) -- pure black, as specified in the design tokens. Not "dark enough." Not "looks black." Computationally verified pure black.

The suite covers 7 action types: navigate (HTTP status checks), screenshot (visual capture), waitForSelector (DOM presence), assert (visibility), evaluate (arbitrary JS assertions), click (interaction), and fill (form input). Here is a form interaction check:

  // From: validation/puppeteer-checks.js

  {
    name: 'login-form-fill',
    route: '/auth/login',
    actions: [
      { type: 'navigate', expected: 200 },
      { type: 'fill', selector: '[data-testid="email-input"]', value: 'test@example.com' },
      { type: 'fill', selector: '[data-testid="password-input"]', value: 'password123' },
      { type: 'screenshot', screenshot: 'login-filled.png' },
    ],
  },

And here is one that validates the admin dashboard has at least 20 tabs:

  // From: validation/puppeteer-checks.js

  {
    name: 'admin-tab-count',
    route: '/admin',
    actions: [
      { type: 'navigate', expected: 200 },
      { type: 'waitForSelector', selector: '[data-testid="admin-tabs"]' },
      { type: 'evaluate', expected: () => document.querySelectorAll('[data-testid^="tab-"]').length >= 20 },
    ],
  },

The validation runner executes checks sequentially (for deterministic screenshots), reports pass/fail with timing, and writes a JSON report:

  // From: validation/run-validation.js

  for (let i = 0; i < filteredChecks.length; i++) {
    const check = filteredChecks[i];
    const progress = `[${String(i + 1).padStart(2, '0')}/${filteredChecks.length}]`;
    process.stdout.write(`  ${progress} ${check.name.padEnd(40, '.')} `);

    const result = await runCheck(browser, check);
    results.push(result);

    if (result.status === 'pass') {
      process.stdout.write(`\x1b[32mPASS\x1b[0m (${result.duration}ms)\n`);
    } else {
      process.stdout.write(`\x1b[31mFAIL\x1b[0m (${result.duration}ms)\n`);
    }
  }

The distribution across screen groups:

  Public screens (Home, Resources, Search, About, Categories x2, Resource Detail): 36 checks
  Auth screens (Login, Register, Forgot Password): 14 checks
  User screens (Profile, Bookmarks, Favorites, History): 18 checks
  Admin screens (Admin Dashboard with 20 tabs, Suggest Edit): 29 checks
  Legal screens (Privacy Policy, Terms of Service): 8 checks
  Total: 105 checks across 107 actions (some checks validate multiple assertions)

The Admin Dashboard alone has 25 checks -- one per tab plus KPI card validation. When you have a 20-tab admin interface, you cannot visually verify all 20 tabs by hand every time you make a change. The Puppeteer suite clicks each tab, waits for the content to render, and screenshots the result. Every time.


THE BRANDING BUG

Here is the most instructive failure from the entire project. During generation, Stitch MCP started producing screens with the product name "Awesome Video Dashboard" instead of the correct name "Awesome Lists." The substitution happened at screen 1 and propagated to 8 screens before it was caught.

The root cause analysis from the branding checklist:

  // From: docs/branding-checklist.md

  ## Why It Happens in AI Workflows

  ### 1. Context Window Drift
  When generating multiple screens in a long session (21+ screens),
  the AI model's "attention" to early instructions diminishes. The
  product name specified in the first prompt may be remembered less
  faithfully by the 15th screen generation.

  ### 2. Training Data Default Patterns
  AI models have seen millions of example UIs with placeholder names
  like "Awesome App," "My Dashboard," "Video Platform." When the
  specific product name isn't prominent in the prompt, the model
  falls back to these common patterns.

  ### 3. Semantic Similarity Substitution
  "Awesome Lists" and "Awesome Video Dashboard" share the word
  "Awesome." The model sometimes pattern-matches on the adjective
  and completes with a more "common" noun phrase from its training data.

The fix is treating branding as a first-class design token, not as free text scattered across prompts. The branding checklist proposes adding brand tokens to the design system:

  // From: docs/branding-checklist.md

  {
    "brand": {
      "name": "Awesome Lists",
      "tagline": "The Ultimate Curated Resource Directory",
      "domain": "awesomeLists.dev",
      "copyrightHolder": "Nick Krzemienski"
    }
  }

And then injecting them programmatically via a prompt builder, so the correct product name is never dependent on human memory or AI attention span.

The automated prevention is a simple grep:

  grep -rn "Video Dashboard\|Placeholder\|lorem ipsum" src/ components/ app/

Run this after every generation session. Add it as a prebuild script. The branding bug is a procedural problem, not a technical one. The AI is perfectly capable of using the correct product name -- it just needs to be told explicitly, every time, in every prompt.


THE PROMPT ENGINEERING PATTERN

Every screen prompt follows the same structure: product name in the first sentence, full design system spec inline, layout description, and key elements list. This pattern worked for all 21 screens:

  Design a [SCREEN NAME] for "Your Product" -- [one-line description].

  DESIGN SYSTEM:
  - Background: #000000 (pure black)
  - Primary Accent: #e050b0 (hot pink)
  - Secondary Accent: #4dacde (cyan)
  - Surface/Card: #111111 background, #1a1a1a elevated
  - Text: #ffffff primary, #a0a0a0 secondary
  - Font: JetBrains Mono, monospaced, used everywhere
  - Border radius: 0px -- brutalist aesthetic
  - Component library: shadcn/ui
  - Borders: 1px solid #2a2a2a

  LAYOUT:
  [Describe structure here]

  KEY ELEMENTS:
  [List all UI elements with specifics]

The redundancy is intentional. Yes, the design system spec is identical across all 21 prompts. Yes, it increases token usage. But it eliminates context window drift entirely. The 15th screen generation has the same design fidelity as the 1st because it has the same context.


THE NUMBERS

  21 screens generated from text descriptions
  47 design tokens governing every visual decision
  107 Puppeteer validation actions (374 individual assertions within those actions)
  5 shadcn/ui component primitives
  4 example compositions
  8 button variants, 8 button sizes
  0 Figma files opened
  0 lines of hand-written CSS
  ~13,432 lines of session transcript
  1 branding bug caught, documented, and solved with a grep script


WHAT THE REMAINING CHALLENGES LOOK LIKE

The interesting conclusion from this project: the remaining challenges are procedural, not technical.

The AI can generate production-quality React components from text descriptions. It can apply design tokens consistently. It can produce 20-tab admin dashboards with proper data-testid attributes. The technology works.

What does not work yet is the process around the technology. Branding drift requires explicit prevention. Long sessions require redundant context. Validation requires deliberate data-testid architecture planned before generation, not added after. These are workflow problems, not capability problems.

The gap between "AI can generate UI" and "AI reliably generates the right UI" is entirely a prompt engineering and validation infrastructure gap. Stitch MCP handles the generation. Puppeteer handles the validation. The hard part -- the part this repo documents -- is everything in between: the token system that constrains the design space, the prompt structure that ensures consistency, the branding safeguards that prevent drift, and the validation suite that proves correctness.

This is the final post in the series. Ten posts. Ten lessons. 8,481 sessions. The tools are here. The patterns are documented. The companion repos have real code you can run today.

The question is no longer whether AI can build production software. It is whether you have the workflows to make it do so reliably.

Companion repo: github.com/krzemienski/stitch-design-to-code

#AgenticDevelopment #StitchMCP #DesignToCode #ReactComponents #PuppeteerValidation
