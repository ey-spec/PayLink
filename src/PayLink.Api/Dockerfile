# ---------- Stage 1: build ----------
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src

# Copy only what restore needs, so this layer stays cached until dependencies change
COPY global.json PayLink.slnx ./
COPY src/PayLink.Api/PayLink.Api.csproj src/PayLink.Api/
RUN dotnet restore src/PayLink.Api/PayLink.Api.csproj

# Now copy the real source and publish
COPY src/PayLink.Api/ src/PayLink.Api/
RUN dotnet publish src/PayLink.Api/PayLink.Api.csproj -c Release -o /app/publish --no-restore

# ---------- Stage 2: run ----------
FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS runtime
WORKDIR /app
COPY --from=build /app/publish .

EXPOSE 8080

# Run as a non-root user (the image provides one)
USER $APP_UID

ENTRYPOINT ["dotnet", "PayLink.Api.dll"]